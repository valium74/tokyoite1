# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A standalone FX Trading Dashboard — a single-file HTML/CSS/JS web page (`index.html`, ~1800 lines). No build step, no dependencies beyond Plotly.js loaded from CDN.

## Architecture

Everything lives in `index.html` with three sections:
1. **CSS** — dark trading terminal theme (`#0e1117` bg, CSS custom properties in `:root`)
2. **HTML** — static layout with pre-structured DOM IDs that JS targets for updates
3. **JavaScript** — all logic inline at the bottom of `<body>`

### JS Structure (inside `<script>`)

| Section | What it does |
|---|---|
| Config constants | `PAIRS`, `USD_QUOTE`, `SESSIONS`, `TIMEFRAMES` |
| State object | `state` — selected pair, timeframe, slider values |
| Cache objects | `histCache` (keyed `base:quote:days`), `corrCache` |
| Seeded PRNG | `mulberry32(seed)` — used to synthesise OHLCV from close-only API data |
| Math helpers | `rollingMean`, `rollingStd`, `ema`, `computeRSI`, `computeMACD`, `computeBBands`, `computeATR` |
| API layer | `fetchLiveRates()`, `fetchHistory(base, quote, days)`, `getHistoryForPair(pair, days)`, `pairPrice(rates, pair)` |
| Render functions | `renderSessions`, `renderLivePrices`, `renderChart`, `renderPriceStats`, `renderTechReadings`, `renderCorrelation`, `renderCorrChart`, `updatePipCalc` |
| Event wiring + init | Bottom of script — `init()` called on `DOMContentLoaded` |

### Data Flow

```
init()
  ├─ fetchLiveRates()        → renderLivePrices()
  ├─ getHistoryForPair()     → renderChart() → Plotly.react()
  │                            renderPriceStats()
  │                            renderTechReadings()
  │                            updatePipCalc()
  └─ renderCorrelation()     → 8× getHistoryForPair(pair, 30) → Plotly.newPlot()
```

Slider/toggle changes: skip API, call `renderChart()` + stats only.
Pair/timeframe changes: re-fetch history (cache hit if revisited), skip correlation re-fetch.
Refresh button: clears `histCache` + `corrCache`, full re-fetch.

### Key Conventions

- **Pair price logic**: `USD_QUOTE` pairs (EUR/USD, GBP/USD, AUD/USD, NZD/USD) store rates as `USD/base`, so display price = `1/rate`. Cross pairs = `rates[quote] / rates[base]`.
- **OHLCV synthesis**: Frankfurter API returns daily closes only. Open/High/Low are synthesised using `mulberry32(42)` seeded RNG + rolling 5-period ATR-like pct. After inverting USD_QUOTE prices, highs and lows must be swapped.
- **Subplots**: Single shared `xaxis`, three y-axes with domains (y: 45–100%, y2: 23–43%, y3: 0–21%). Shapes referencing `yref:'y2'` for RSI lines.
- **Decimal places**: 3 for JPY pairs, 5 for all others — checked with `pair.includes("JPY")`.

## External APIs (public, CORS-enabled)

- Live rates: `https://open.er-api.com/v6/latest/USD` (TTL ~60s)
- History: `https://api.frankfurter.app/{start}..{end}?from={base}&to={quote}` (TTL ~1h)

## Local Development

No build step. Serve `index.html` directly:

```bash
python3 -m http.server 8000
# then open http://localhost:8000
```

## Deployment

Hosted on **AWS Amplify** — app name `tokyoite1`, connected to GitHub repo `https://github.com/valium74/tokyoite1`.

Deployments are triggered automatically by pushing to the `main` branch:

```bash
git add <files>
git commit -m "your message"
git push origin main
```

No build step — Amplify serves the static files directly.
