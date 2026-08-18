# Massive API Reference

Reference documentation for the Massive REST API (formerly Polygon.io), as used by FinAlly's real-market-data path.

## Overview

- **Company**: Polygon.io rebranded as **Massive.com** on Oct 30, 2025. Existing API keys and integrations keep working unmodified.
- **Base URL**: `https://api.massive.com` (the legacy `https://api.polygon.io` host is still accepted for backward compatibility, but new code should not rely on it)
- **Python package**: `massive` (formerly `polygon-api-client`) — install with `pip install -U massive` / `uv add massive`
- **Min Python version**: 3.9+
- **Auth**: API key via `MASSIVE_API_KEY` env var, or passed explicitly to `RESTClient(api_key=...)`. The client sends it as a bearer token automatically — application code never builds the `Authorization` header by hand.
- **Debugging**: `RESTClient(trace=True, verbose=True)` logs outgoing requests/responses.

## Rate Limits & Snapshot Ticker Limits

| Tier | Request rate | Notes |
|------|---------------|-------|
| Free | 5 requests/minute | Plenty for a single polling loop if the interval is long enough |
| Paid (all tiers) | Much higher, tier-dependent | Recommended to stay well under any published ceiling |

Separately, the **Unified Snapshot** endpoint accepts a maximum of **250 symbols per request**. FinAlly's default watchlist (10 tickers) and any reasonable growth of it stays far under this, so one call per poll cycle is always enough — there is no need to batch across multiple requests.

For FinAlly we poll on a fixed timer rather than using rate-limit response headers to self-tune:
- **Free tier**: poll every 15s (5 req/min ⇒ max 1 req every 12s; 15s leaves headroom)
- **Paid tiers**: poll every 2–5s

## Client Initialization

```python
from massive import RESTClient

# Reads MASSIVE_API_KEY from the environment automatically if omitted
client = RESTClient()

# Or pass explicitly
client = RESTClient(api_key="your_key_here")
```

The client is **synchronous**. In an asyncio application (FastAPI), calls must be run off the event loop, e.g. via `asyncio.to_thread(...)`.

## Endpoints Used in FinAlly

### 1. Snapshot — Multiple Tickers (primary endpoint)

Gets current price/quote/trade data for many tickers in a **single API call** — this is the endpoint the market-data poller uses every cycle.

**REST**: `GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,GOOGL,MSFT`

**Python client**:
```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient()

snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA"],
)

for snap in snapshots:
    print(f"{snap.ticker}: ${snap.last_trade.price}")
    print(f"  Day change: {snap.day.change_percent}%")
    print(f"  Day OHLC: O={snap.day.open} H={snap.day.high} L={snap.day.low} C={snap.day.close}")
    print(f"  Volume: {snap.day.volume}")
```

**Response shape** (per ticker):
```json
{
  "ticker": "AAPL",
  "day": {
    "open": 129.61,
    "high": 130.15,
    "low": 125.07,
    "close": 125.07,
    "volume": 111237700,
    "volume_weighted_average_price": 127.35,
    "previous_close": 129.61,
    "change": -4.54,
    "change_percent": -3.50
  },
  "last_trade": {
    "price": 125.07,
    "size": 100,
    "exchange": "XNYS",
    "timestamp": 1675190399000
  },
  "last_quote": {
    "bid_price": 125.06,
    "ask_price": 125.08,
    "bid_size": 500,
    "ask_size": 1000,
    "spread": 0.02,
    "timestamp": 1675190399500
  },
  "prev_daily_bar": { "...": "previous day OHLCV" },
  "minute_volume": { "...": "volume for the current minute" }
}
```

**Fields FinAlly actually reads** (see `backend/app/market/massive_client.py`):
- `snap.ticker` — cache key
- `snap.last_trade.price` — the current price written to the cache
- `snap.last_trade.timestamp` — Unix **milliseconds**; divide by 1000 before storing (the cache and `PriceUpdate` model use Unix seconds)

Note: FinAlly's poller intentionally does **not** use `day.previous_close` — the price cache computes its own `previous_price` from the last cached value, so day-open reference prices aren't needed for the live-tick display. `day.change_percent` and the OHLC fields are available if a future feature (e.g. a detail panel) wants them.

### 2. Single Ticker Snapshot

For a detail view of one ticker (e.g. the user clicks a row in the watchlist).

```python
snapshot = client.get_snapshot_ticker(
    market_type=SnapshotMarketType.STOCKS,
    ticker="AAPL",
)

print(f"Price: ${snapshot.last_trade.price}")
print(f"Bid/Ask: ${snapshot.last_quote.bid_price} / ${snapshot.last_quote.ask_price}")
print(f"Day range: ${snapshot.day.low} - ${snapshot.day.high}")
```

Not currently wired into any FinAlly endpoint — noted here because it's the natural next call if a richer per-ticker detail view is added later.

### 3. Previous Close

Previous trading day's OHLC for a ticker. Useful for seeding realistic starting prices for a ticker the user adds that isn't in the built-in seed list.

**REST**: `GET /v2/aggs/ticker/{ticker}/prev`

```python
prev = client.get_previous_close_agg(ticker="AAPL")

for agg in prev:
    print(f"Previous close: ${agg.close}")
    print(f"OHLC: O={agg.open} H={agg.high} L={agg.low} C={agg.close}")
    print(f"Volume: {agg.volume}")
```

```json
{
  "ticker": "AAPL",
  "results": [
    {"o": 150.0, "h": 155.0, "l": 149.0, "c": 154.5, "v": 1000000, "t": 1672531200000}
  ]
}
```

### 4. Aggregates (Bars)

Historical OHLCV bars over a date range. Not needed for live polling; the natural source for a historical chart feature if one is added.

**REST**: `GET /v2/aggs/ticker/{ticker}/range/{multiplier}/{timespan}/{from}/{to}`

```python
aggs = []
for a in client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="day",
    from_="2024-01-01",
    to="2024-01-31",
    limit=50000,
):
    aggs.append(a)

for a in aggs:
    print(f"Date: {a.timestamp}, O={a.open} H={a.high} L={a.low} C={a.close} V={a.volume}")
```

```json
{"o": 130.0, "h": 132.5, "l": 129.8, "c": 131.2, "v": 50000000, "t": 1672531200000}
```

### 5. Last Trade / Last Quote

Single-purpose endpoints if only the most recent trade or NBBO quote is needed, without the rest of the snapshot payload.

```python
trade = client.get_last_trade(ticker="AAPL")
print(f"Last trade: ${trade.price} x {trade.size}")

quote = client.get_last_quote(ticker="AAPL")
print(f"Bid: ${quote.bid} x {quote.bid_size}")
print(f"Ask: ${quote.ask} x {quote.ask_size}")
```

## How FinAlly Uses the API

`MassiveDataSource` (`backend/app/market/massive_client.py`) runs as a background asyncio task:

1. Collects the current ticker list from the watchlist (mutated live via `add_ticker`/`remove_ticker`)
2. Calls `get_snapshot_all()` with those tickers — one API call per cycle, run via `asyncio.to_thread` since the client is synchronous
3. Extracts `last_trade.price` and `last_trade.timestamp` (converted ms → seconds) from each snapshot
4. Writes each result into the shared `PriceCache`
5. Sleeps for the poll interval (default 15s), then repeats

```python
import asyncio
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

async def poll_massive(api_key: str, get_tickers, price_cache, interval: float = 15.0):
    """Poll Massive and update the price cache. Mirrors MassiveDataSource._poll_loop."""
    client = RESTClient(api_key=api_key)

    while True:
        tickers = get_tickers()
        if tickers:
            snapshots = await asyncio.to_thread(
                client.get_snapshot_all,
                market_type=SnapshotMarketType.STOCKS,
                tickers=tickers,
            )
            for snap in snapshots:
                price_cache.update(
                    ticker=snap.ticker,
                    price=snap.last_trade.price,
                    timestamp=snap.last_trade.timestamp / 1000.0,
                )

        await asyncio.sleep(interval)
```

## Error Handling

- **401** — invalid API key
- **403** — plan doesn't include the requested endpoint
- **429** — rate limit exceeded (free tier: 5 req/min)
- **5xx** — server errors; the client retries a few times internally

FinAlly's poller wraps each cycle in a broad `try/except`, logs the failure, and does **not** re-raise — a failed poll just means stale prices persist in the cache until the next successful cycle. This matches the "no confirmation dialog, keep the demo flowing" philosophy: a transient Massive outage shouldn't crash the app.

## Notes

- The snapshot endpoint returns data for **all requested tickers in one call** (up to 250) — critical for staying within rate limits, especially on the free tier
- Timestamps from the API are Unix **milliseconds**; FinAlly's internal `PriceUpdate`/`PriceCache` use Unix **seconds**, so every value crossing that boundary must be divided by 1000
- During closed-market hours, `last_trade.price` reflects the last traded price (which may be from after-hours or the prior session)
- The `day` object resets at market open; during pre-market it may still reflect the previous session
