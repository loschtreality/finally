# Market Simulator Design

Approach and code structure for simulating realistic stock prices when no `MASSIVE_API_KEY` is configured — the default mode most users run in. Implemented in `backend/app/market/simulator.py` and `backend/app/market/seed_prices.py`.

## Overview

The simulator uses **Geometric Brownian Motion (GBM)** — the standard model underlying Black-Scholes option pricing. Prices evolve continuously with random noise, are guaranteed positive (the update is multiplicative via `exp()`), and produce the lognormal return distribution seen in real markets.

`SimulatorDataSource` runs `GBMSimulator.step()` on an asyncio loop every 500ms and writes the results to the shared `PriceCache` (see `market_interface.md`).

## GBM Math

At each time step, a price evolves as:

```
S(t+dt) = S(t) * exp((mu - sigma^2/2) * dt + sigma * sqrt(dt) * Z)
```

- `S(t)` — current price
- `mu` — annualized drift (expected return), e.g. `0.05` (5%/year)
- `sigma` — annualized volatility, e.g. `0.20` (20%/year)
- `dt` — time step as a fraction of a trading year
- `Z` — standard normal random draw (correlated across tickers — see below)

For 500ms updates over a 252-day, 6.5-hour trading year:

```python
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # 5,896,800
DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR   # ~8.48e-8
```

This tiny `dt` produces small, sub-cent-scale moves per tick that accumulate into realistic intraday ranges over minutes and hours — a stock doesn't jump 5% in one 500ms frame, it drifts there over thousands of frames.

## Correlated Moves

Real stocks don't move independently — tech names tend to move together, etc. The simulator uses a **Cholesky decomposition** of a correlation matrix to turn independent normal draws into correlated ones:

```
L = cholesky(C)              # C = correlation matrix, positive semi-definite
Z_correlated = L @ Z_independent
```

Correlation structure (`seed_prices.py` / `_pairwise_correlation`):

| Group | Members | Correlation |
|---|---|---|
| Tech (intra-group) | AAPL, GOOGL, MSFT, AMZN, META, NVDA, NFLX | 0.6 |
| Finance (intra-group) | JPM, V | 0.5 |
| TSLA vs. anything | — | 0.3 (does its own thing, even though it's grouped with tech) |
| Cross-sector / unknown | everything else | 0.3 |

The Cholesky decomposition is rebuilt whenever a ticker is added or removed (`_rebuild_cholesky`) — O(n²), acceptable since the watchlist is small (well under 50 tickers even with generous ad-hoc additions).

## Random Events

Each tick, each ticker independently has a small chance (`event_probability`, default `0.001` = 0.1%) of a sudden 2–5% shock, for visual drama on the dashboard:

```python
if random.random() < self._event_prob:
    shock_magnitude = random.uniform(0.02, 0.05)
    shock_sign = random.choice([-1, 1])
    self._prices[ticker] *= 1 + shock_magnitude * shock_sign
```

At 2 ticks/sec with 10 tickers, expect roughly one event across the whole watchlist every ~50 seconds — enough to keep the screen interesting without every move looking like an "event."

## Seed Prices & Per-Ticker Parameters

`seed_prices.py` holds only constant data — no logic:

```python
SEED_PRICES: dict[str, float] = {
    "AAPL": 190.00, "GOOGL": 175.00, "MSFT": 420.00, "AMZN": 185.00,
    "TSLA": 250.00, "NVDA": 800.00, "META": 500.00, "JPM": 195.00,
    "V": 280.00, "NFLX": 600.00,
}

TICKER_PARAMS: dict[str, dict[str, float]] = {
    "AAPL":  {"sigma": 0.22, "mu": 0.05},
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT":  {"sigma": 0.20, "mu": 0.05},
    "AMZN":  {"sigma": 0.28, "mu": 0.05},
    "TSLA":  {"sigma": 0.50, "mu": 0.03},   # high volatility
    "NVDA":  {"sigma": 0.40, "mu": 0.08},   # high volatility, strong drift
    "META":  {"sigma": 0.30, "mu": 0.05},
    "JPM":   {"sigma": 0.18, "mu": 0.04},   # low volatility (bank)
    "V":     {"sigma": 0.17, "mu": 0.04},   # low volatility (payments)
    "NFLX":  {"sigma": 0.35, "mu": 0.05},
}

DEFAULT_PARAMS: dict[str, float] = {"sigma": 0.25, "mu": 0.05}

CORRELATION_GROUPS: dict[str, set[str]] = {
    "tech": {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}
INTRA_TECH_CORR = 0.6
INTRA_FINANCE_CORR = 0.5
CROSS_GROUP_CORR = 0.3
TSLA_CORR = 0.3
```

A ticker added dynamically (via the watchlist UI or the AI chat, not in `SEED_PRICES`) starts at a uniform-random price between $50–$300 and uses `DEFAULT_PARAMS` for volatility/drift — plausible enough for a demo without needing a real reference price.

## Implementation

```python
import math, random
import numpy as np

class GBMSimulator:
    """Geometric Brownian Motion simulator for correlated stock prices."""

    TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600
    DEFAULT_DT = 0.5 / TRADING_SECONDS_PER_YEAR

    def __init__(self, tickers, dt=DEFAULT_DT, event_probability=0.001):
        self._dt = dt
        self._event_prob = event_probability
        self._tickers, self._prices, self._params = [], {}, {}
        self._cholesky = None
        for ticker in tickers:
            self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def step(self) -> dict[str, float]:
        """Advance all tickers by one tick. Hot path — called every 500ms."""
        n = len(self._tickers)
        if n == 0:
            return {}

        z_independent = np.random.standard_normal(n)
        z = self._cholesky @ z_independent if self._cholesky is not None else z_independent

        result = {}
        for i, ticker in enumerate(self._tickers):
            mu, sigma = self._params[ticker]["mu"], self._params[ticker]["sigma"]
            drift = (mu - 0.5 * sigma**2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z[i]
            self._prices[ticker] *= math.exp(drift + diffusion)

            if random.random() < self._event_prob:
                shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
                self._prices[ticker] *= 1 + shock

            result[ticker] = round(self._prices[ticker], 2)
        return result

    def add_ticker(self, ticker):
        if ticker in self._prices:
            return
        self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def remove_ticker(self, ticker):
        if ticker not in self._prices:
            return
        self._tickers.remove(ticker)
        del self._prices[ticker]
        del self._params[ticker]
        self._rebuild_cholesky()

    def get_price(self, ticker):
        return self._prices.get(ticker)

    def get_tickers(self):
        return list(self._tickers)

    def _add_ticker_internal(self, ticker):
        if ticker in self._prices:
            return
        self._tickers.append(ticker)
        self._prices[ticker] = SEED_PRICES.get(ticker, random.uniform(50.0, 300.0))
        self._params[ticker] = TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))

    def _rebuild_cholesky(self):
        n = len(self._tickers)
        if n <= 1:
            self._cholesky = None
            return
        corr = np.eye(n)
        for i in range(n):
            for j in range(i + 1, n):
                rho = self._pairwise_correlation(self._tickers[i], self._tickers[j])
                corr[i, j] = corr[j, i] = rho
        self._cholesky = np.linalg.cholesky(corr)

    @staticmethod
    def _pairwise_correlation(t1, t2):
        tech, finance = CORRELATION_GROUPS["tech"], CORRELATION_GROUPS["finance"]
        if t1 == "TSLA" or t2 == "TSLA":
            return TSLA_CORR
        if t1 in tech and t2 in tech:
            return INTRA_TECH_CORR
        if t1 in finance and t2 in finance:
            return INTRA_FINANCE_CORR
        return CROSS_GROUP_CORR
```

## Wrapping It in a `MarketDataSource`

`SimulatorDataSource` adapts `GBMSimulator` to the `MarketDataSource` interface (`market_interface.md`) via an asyncio background task:

```python
class SimulatorDataSource(MarketDataSource):
    def __init__(self, price_cache, update_interval=0.5, event_probability=0.001):
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._sim = None
        self._task = None

    async def start(self, tickers):
        self._sim = GBMSimulator(tickers=tickers, event_probability=self._event_prob)
        for ticker in tickers:                       # seed cache immediately —
            price = self._sim.get_price(ticker)       # don't wait for the first tick
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")

    async def stop(self):
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        self._task = None

    async def add_ticker(self, ticker):
        if self._sim:
            self._sim.add_ticker(ticker)
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)

    async def remove_ticker(self, ticker):
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)

    def get_tickers(self):
        return self._sim.get_tickers() if self._sim else []

    async def _run_loop(self):
        while True:
            try:
                if self._sim:
                    for ticker, price in self._sim.step().items():
                        self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)
```

The `try/except Exception` inside the loop (not around it) is deliberate: a single bad step logs and gets skipped, but the loop itself — and the `asyncio.sleep` cadence — keeps running. A crashed background task would otherwise silently stop all price updates for the rest of the session.

## File Structure

```
backend/
  app/
    market/
      simulator.py       # GBMSimulator + SimulatorDataSource
      seed_prices.py       # SEED_PRICES, TICKER_PARAMS, DEFAULT_PARAMS, correlation constants
```

`seed_prices.py` is pure data (no functions/classes) so it can be tuned — e.g. adding a new default ticker, adjusting a sigma — without touching any simulation logic.

## Behavior Notes

- Prices never go negative — GBM's multiplicative `exp()` update guarantees positivity regardless of how extreme `Z` or the random-event shock is
- The tiny `dt` produces sub-cent moves per tick; realistic-looking intraday ranges emerge from accumulation over many ticks, not from any single big jump
- `sigma=0.50` (TSLA) over a simulated trading day produces roughly the intraday range you'd expect from a genuinely volatile stock
- The correlation matrix is built to be positive semi-definite by construction (values are ≤ 0.6, structured by group), which is what Cholesky decomposition requires — if tuning these constants, keep any set of pairwise correlations mathematically consistent (e.g. don't set three tickers all pairwise-correlated at 0.99 while also imposing a conflicting constraint)
- Adding/removing a ticker mid-session rebuilds the Cholesky matrix (`O(n²)`), which is a foreground synchronous call, not scheduled work — negligible at watchlist scale (n < 50)
- A Rich terminal demo (`backend/market_data_demo.py`, `uv run market_data_demo.py`) renders a live dashboard of all tracked tickers with sparklines and an event log — the fastest way to visually sanity-check parameter changes without running the full stack
