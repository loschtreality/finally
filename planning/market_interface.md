# Market Data Interface Design

Unified interface for market data in FinAlly. Two implementations — `SimulatorDataSource` and `MassiveDataSource` — sit behind one abstract interface (`MarketDataSource`) so all downstream code (SSE streaming, trade execution, portfolio valuation) is source-agnostic. Selection between them is driven entirely by whether `MASSIVE_API_KEY` is set (see `massive_API.md` for the Massive side, `market_simulator.md` for the simulator side).

This reflects the implementation in `backend/app/market/` (see `planning/MARKET_DATA_SUMMARY.md` for the shipped-and-tested summary).

## Core Data Model

```python
from dataclasses import dataclass, field
import time

@dataclass(frozen=True, slots=True)
class PriceUpdate:
    """Immutable snapshot of a single ticker's price at a point in time."""

    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds

    @property
    def change(self) -> float:
        return round(self.price - self.previous_price, 4)

    @property
    def change_percent(self) -> float:
        if self.previous_price == 0:
            return 0.0
        return round((self.price - self.previous_price) / self.previous_price * 100, 4)

    @property
    def direction(self) -> str:
        if self.price > self.previous_price:
            return "up"
        elif self.price < self.previous_price:
            return "down"
        return "flat"

    def to_dict(self) -> dict:
        """Serialize for JSON / SSE transmission."""
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": self.change_percent,
            "direction": self.direction,
        }
```

`change`, `change_percent`, and `direction` are **computed properties**, not stored fields — there's a single source of truth (`price` vs `previous_price`) and no risk of them drifting out of sync. `PriceUpdate` is the only data structure that leaves the market data layer; everything downstream (SSE payloads, trade pricing, portfolio math) works with it or its `to_dict()` form.

## Abstract Interface

```python
from abc import ABC, abstractmethod

class MarketDataSource(ABC):
    """Contract for market data providers.

    Implementations push price updates into a shared PriceCache on their own
    schedule. Downstream code never calls the data source directly for prices —
    it reads from the cache.
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates. Starts a background task. Call once."""

    @abstractmethod
    async def stop(self) -> None:
        """Stop the background task and release resources. Safe to call twice."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker to the active set. No-op if already present."""

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker from the active set. Also removes it from the cache."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return the current list of actively tracked tickers."""
```

The interface does **not** return prices directly from any method — both implementations push updates into a shared cache on their own schedule (500ms for the simulator, 15s default for Massive). This is what makes the two sources interchangeable despite very different update cadences: consumers always read "whatever's freshest" from the cache rather than being coupled to either source's timing.

## Price Cache

The single point of truth that both sources write to and everything else reads from.

```python
import time
from threading import Lock

class PriceCache:
    """Thread-safe in-memory cache of the latest price for each ticker.

    Writers: SimulatorDataSource or MassiveDataSource (one at a time).
    Readers: SSE streaming endpoint, portfolio valuation, trade execution.
    """

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()
        self._version: int = 0  # bumped on every update; drives SSE change detection

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        with self._lock:
            ts = timestamp or time.time()
            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else price  # first update: direction="flat"

            update = PriceUpdate(
                ticker=ticker,
                price=round(price, 2),
                previous_price=round(previous_price, 2),
                timestamp=ts,
            )
            self._prices[ticker] = update
            self._version += 1
            return update

    def get(self, ticker: str) -> PriceUpdate | None:
        with self._lock:
            return self._prices.get(ticker)

    def get_all(self) -> dict[str, PriceUpdate]:
        with self._lock:
            return dict(self._prices)  # shallow copy — safe snapshot for readers

    def get_price(self, ticker: str) -> float | None:
        update = self.get(ticker)
        return update.price if update else None

    def remove(self, ticker: str) -> None:
        with self._lock:
            self._prices.pop(ticker, None)

    @property
    def version(self) -> int:
        return self._version
```

Key points:
- A plain `threading.Lock` is enough even though the app is asyncio-based, because both the simulator loop and the Massive poller run in the same process and cache reads/writes are short, non-blocking dict operations.
- `version` is a monotonic counter the SSE endpoint polls cheaply to detect "has anything changed since I last sent a payload" without diffing dictionaries.

## Factory Function

Selects the implementation at startup, purely from environment:

```python
import os

def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """MASSIVE_API_KEY set and non-empty -> MassiveDataSource. Otherwise -> SimulatorDataSource.

    Returns an unstarted source; caller must await source.start(tickers).
    """
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()

    if api_key:
        from .massive_client import MassiveDataSource
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        from .simulator import SimulatorDataSource
        return SimulatorDataSource(price_cache=price_cache)
```

No other code branches on `MASSIVE_API_KEY` — this factory is the single decision point, matching the PLAN.md requirement that "all downstream code ... is agnostic to the source."

## Implementation Summaries

Both are covered in depth elsewhere (`massive_API.md`, `market_simulator.md`); the shape that matters for the interface:

- **`MassiveDataSource`** — on `start()`, does an immediate poll (so the cache isn't empty while waiting for the first interval to elapse), then loops: sleep, poll, repeat. `add_ticker`/`remove_ticker` just mutate the in-memory ticker list; the next poll cycle picks up the change. The Massive REST client is synchronous, so every poll runs via `asyncio.to_thread` to avoid blocking the event loop.
- **`SimulatorDataSource`** — on `start()`, constructs a `GBMSimulator` and immediately seeds the cache with starting prices (so the UI has data before the first tick), then loops every `update_interval` (default 500ms): step the simulator, write each ticker's new price to the cache. `add_ticker`/`remove_ticker` delegate to the simulator (which rebuilds its correlation matrix) and seed/clear the cache accordingly.

Both wrap their per-cycle work in `try/except` so a single bad tick or failed poll never kills the background task — it logs and continues on the next cycle.

## Integration with SSE

`backend/app/market/stream.py` reads from the cache and streams to the browser via `EventSource`:

```python
@router.get("/prices")
async def stream_prices(request: Request) -> StreamingResponse:
    return StreamingResponse(
        _generate_events(price_cache, request),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",
        },
    )

async def _generate_events(price_cache, request, interval=0.5):
    yield "retry: 1000\n\n"  # tell EventSource to reconnect after 1s if dropped

    last_version = -1
    try:
        while True:
            if await request.is_disconnected():
                break
            current_version = price_cache.version
            if current_version != last_version:
                last_version = current_version
                prices = price_cache.get_all()
                if prices:
                    data = {ticker: update.to_dict() for ticker, update in prices.items()}
                    yield f"data: {json.dumps(data)}\n\n"
            await asyncio.sleep(interval)
    except asyncio.CancelledError:
        pass
```

Notes on this design:
- Polling `price_cache.version` rather than diffing dictionaries keeps each loop iteration cheap regardless of watchlist size.
- Because it only emits a payload when `version` changed, an idle cache (e.g. Massive poller between cycles) doesn't spam the client with duplicate frames — but the client still gets a full snapshot (`{ticker: {...}, ...}` for every tracked ticker) on each emitted frame, not a delta, which keeps client-side state trivial to reconstruct after a reconnect.
- `request.is_disconnected()` plus the `CancelledError` handler are what let many browser tabs open/close SSE connections without leaking server-side tasks.

## File Structure

```
backend/
  app/
    market/
      __init__.py
      models.py             # PriceUpdate
      interface.py           # MarketDataSource ABC
      cache.py                # PriceCache
      factory.py              # create_market_data_source()
      massive_client.py        # MassiveDataSource
      simulator.py             # GBMSimulator + SimulatorDataSource
      seed_prices.py            # SEED_PRICES, TICKER_PARAMS, correlation constants
      stream.py                 # create_stream_router() — SSE endpoint factory
```

## Lifecycle

1. **App startup**: create a `PriceCache`, call `create_market_data_source(price_cache)`, then `await source.start(initial_tickers)`
2. **Watchlist changes**: `await source.add_ticker(ticker)` / `await source.remove_ticker(ticker)` — take effect on the next update cycle
3. **SSE streaming**: `create_stream_router(price_cache)` mounted at `/api/stream/prices`, pushing full snapshots on change, checked every 500ms
4. **Trade execution**: reads the current price via `price_cache.get_price(ticker)` — market orders fill at whatever's currently cached
5. **App shutdown**: `await source.stop()`
