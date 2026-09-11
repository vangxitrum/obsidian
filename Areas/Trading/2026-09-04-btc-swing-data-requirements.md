---
tags: [trading, m5, btc, swing]
created: 2026-09-04
---

# Data Requirements - Low-Risk BTC Swing Style (15-20% TP)

Context: M5 workspace, `binancef btc/usd`, chart timeframe 1h. Goal: hold a futures position
for a long time, take profit around 15-20%, survive drawdown without forced exit.

## 0. The fork that decides everything

- **15-20% of price**: BTC must move +15-20% from entry. Daily ATR ~1.5-2.5% -> typical hold
  2-6 weeks. This is a swing trade; bias must come from 4h/1D, not the 1h chart.
- **15-20% ROI on margin**: at 5x leverage that is only a ~3-4% price move. Hold measured in
  days. Different data, tighter SL, funding matters much less.

Decide this before writing any script. Everything below assumes the first (price-move) case
unless noted.

DECIDED 2026-09-04: 15-20% means ROI on margin, not price move. At 5x leverage that is a 3-4% price move, hold measured in days.

## 1. Volatility budget - `data.OHLCV` (1D)

`ta.atr(14)` expressed as % of price.

    days_to_target ~= target% / (ATR%_per_day * trend_efficiency)
    trend_efficiency ~ 0.3 in chop, ~0.6 trending

Purpose: answer "is 15-20% even reachable in my holding window". Without it you set a TP price
never reaches.

## 2. Structure levels - `data.OHLCV` (4h + 1D)

`ta.pivothigh()` / `ta.pivotlow()` swing highs and lows.
Entry on retest of the nearest level. SL beyond the swing that invalidates the thesis.
SL must be structural, never "2% because it feels safe".

## 3. Volume profile - `data.VOLUME`

POC, VAH, VAL. High-volume nodes act as magnets and support; low-volume gaps travel fast.
Place TP *before* a large opposing node, not inside it.

## 4. Order book - `data.BOOK`

Resting liquidity walls / heatmap. Long holds get wicked into stops parked at obvious round
numbers. Use to avoid crowded stop locations.

## 5. Funding rate - `data.STAT`  **(critical, most ignored)**

Perp funding is paid every 8h.

| Funding per 8h | Monthly carry cost |
|---|---|
| 0.01% (baseline) | ~1.1% |
| 0.05% | ~4.5% |
| 0.10% | ~9% |

A 6-week long through a hot positive-funding regime can eat half of a 15-20% target.
Rule: if funding stays above 0.03% per 8h, wait, or size for the carry.

## 6. Open interest - `data.OI` (read together with price)

- Price up + OI up -> leveraged crowd long, squeeze risk.
- Price up + OI down -> shorts covering, weak follow-through.
- Price up + OI flat + spot CVD up -> real spot buying, the kind that survives weeks.

This distinction separates a hold worth sitting through from one that liquidates you.

## 7. CVD / volume delta - `data.CVD`, `data.VD`

Track spot exchanges and futures separately. Spot CVD leading = healthy trend.
Futures-only CVD = fragile, leverage-driven.

## 8. Liquidations - `data.STAT`

Liquidation clusters mark where the wick goes. Put the SL *below* the cluster, never inside it.

## 9. Position math (not from M5 - from you)

Leverage, liquidation price, risk % per trade, max adverse excursion (MAE) tolerated.
"Safe endure" means: liquidation price is far beyond the SL, and the SL sits at structural
invalidation rather than at a pain threshold.

## Rules that follow

- Risk 0.5-1% of account per trade.
- Leverage 2-3x max for weeks-long holds; liq price must be unreachable before SL triggers
  (target: liq distance >= 3x SL distance).
- R:R >= 3. A 15-20% TP with a 5% SL = 3-4R. Under 2R is not worth weeks of funding.
- Enter only where invalidation is structurally close. If invalidation is 12% away, the setup
  is untradeable at that size.
- Log per setup: entry, SL, TP, R, expected days-to-target, estimated funding cost,
  invalidation condition.

## Implementation

M5 script `ROI Entry Planner` (script id e882d94f-2be0-445c-8db9-318dc7002ba3), attached to the binancef btc/usd chart. Entry logic: swing pivots + ATR-padded stop, EMA-slope direction bias. TP price is derived as targetROI / leverage. It reports verdict TRADEABLE only when loss ROI <= max, liq distance >= 3x SL distance, and R:R >= 3. Funding rate cost is not modelled in the script yet (data.STAT accessors unverified).
