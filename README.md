# ICT / SMC Signal Desk

Pulls live OHLC candles, runs a rule-based ICT/Smart Money Concepts engine
on them (market structure, fair value gaps, order blocks, liquidity pools,
premium/discount), and outputs a trade signal: **BUY / SELL / WAIT** with
entry, stop-loss, take-profit, R:R and a plain-English reasoning chain.

```
backend/   Node/Express API — fetches candles, runs the engine, serves JSON
frontend/  React dashboard — candlestick chart + live signal card
```

## How the engine reads a chart (mirrors the manual ICT checklist)

1. **Structure** — fractal swing highs/lows → BOS (trend continuation) and
   CHoCH (first break against trend = potential reversal), using candle
   closes against the last unbroken swing.
2. **Premium/Discount** — equilibrium of the current dealing range; only
   trades in discount for buys / premium for sells (downgrades confidence
   otherwise).
3. **POI** — nearest untested order block (fallback: fair value gap) formed
   on the current leg, on the correct side of price for a pullback entry.
4. **Stop** — beyond the invalidation swing / POI edge.
5. **Target** — nearest unswept liquidity pool (equal highs/lows) in the
   trade's direction, else the opposite edge of the current range.
6. **Filter** — anything under 1.5 R:R is downgraded to WAIT.

This is the same logic from the checklist, coded up — it will say WAIT far
more often than it says BUY/SELL, which is intentional.

## Quick start

### 1. Backend

```bash
cd backend
npm install
cp .env.example .env     # optional — see below
npm run dev               # http://localhost:4000
```

**No API key needed to try it** — without `TWELVEDATA_API_KEY` set, the
API serves a bundled synthetic candle series (a down-move → sweep →
reversal, similar to a real session) so the whole pipeline runs end to end.

For live gold/forex/crypto data, get a free key at
[twelvedata.com](https://twelvedata.com) (800 requests/day free tier,
covers `XAU/USD`, major FX pairs, and crypto) and put it in `backend/.env`:

```
TWELVEDATA_API_KEY=your_key_here
```

Run `npm run test:engine` any time to sanity-check the engine against the
synthetic series (swings, structure events, FVGs, order blocks, the final
signal) without needing the API key or the frontend.

### 2. Frontend

```bash
cd frontend
npm install
npm run dev               # http://localhost:5173
```

Opens a dashboard with a symbol/interval picker, live candlestick chart
(entry/SL/TP and zones drawn as price lines), and the signal card. Auto
refreshes every 60s.

## Extending

- **More symbols**: edit `SUPPORTED_SYMBOLS` in `backend/.env` — anything
  TwelveData supports (`XAU/USD`, `BTC/USD`, `EUR/USD`, etc.) works as-is.
- **Different data provider**: swap the fetch logic in
  `backend/src/data/priceClient.js` — everything downstream just needs
  chronological `{ time, open, high, low, close }` candles.
- **Tune the engine**: `backend/src/engine/` has one file per concept
  (swings, structure, fvg, orderBlocks, liquidity, premiumDiscount) —
  each is independent and unit-testable on its own.
- **Alerts**: `backend/src/routes/analyze.js` is a plain REST endpoint —
  easy to poll from a cron job / scheduled task and push to
  Telegram/WhatsApp/email when `signal.verdict !== 'WAIT'`.

## Notes

- This is pattern-recognition on price history, not investment advice —
  the UI says so, and the engine is deliberately conservative (WAIT is the
  default outcome unless every step lines up).
- Sandbox note: this project was scaffolded in an environment without
  npm registry access, so `npm install` hasn't been run here — the engine
  logic itself was verified directly with `node src/engine/selftest.js`
  (zero external deps). Running `npm install` in your own environment is
  expected to work normally.
