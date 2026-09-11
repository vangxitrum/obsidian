---
type: fact
tags: [binance, stops, resolver, backtest]
created: 2026-09-10
agent: main
---

# SL cluster 2026-09-09 night / 2026-09-10 morning

BTC slid 79700 -> 77650 overnight in a RANGE regime; RANGE-fade LONG readings stopped out at 03:30 and 05:09 (+07).
Only one raised signal (T-SGLL) actually hit SL; the rest were suppressed repeat readings.

## Stop cause (lib/plan.ts)
- Entry at "value area low" ~78030 sat $10-80 above swing low 78020; stop = nearest level below entry - 0.5 ATR pad = ~77855 (0.5-0.75 ATR). Range low 77600 sat below; wick to 77706 hit between.
- Swing low confirms only after pivotRight=5 bars (2.5h on 30m), so the stop anchored above an already-printed low.
- Fix added 2026-09-10: `freshExtreme` (lowest/highest of last pivotRight+1 bars) is a stop candidate; `minStopLevelAtr` (TRACE_MIN_STOP_LEVEL_ATR) skips levels too close to entry. Regression fixtures: lib/fixtures/T-7SE9.json, T-JDDG.json (lib/plan.incident.test.ts) reproduce the live plan exactly.
- BACKTEST VERDICT (npm run tune -- stop 30m, 2022-2026): baseline 3.16x equity; fresh-low only 3.01x; level>=0.5 2.94x; level>=1 3.09x; any 1 ATR floor 2.4-2.6x. The stop changes do not improve history; a 1 ATR floor is clearly worse. Tight stops + SL clusters are the expected cost of this strategy.

## Resolver bugs (fixed 2026-09-10)
- Old resolver used gapHigh/gapLow of the last hour of 30m bars: no fill check, pre-capture prices counted, and gaps between captures missed touches. Also capture passed `entry` as the close.
- New lib/resolve.ts walks 1m bars after capturedAt, requires fill (Snapshot.filledAt), resumes from Snapshot.checkedThrough, new outcome UNFILLED. Migration 20260910120000_add_snapshot_fill.
- `npm run reresolve` re-settled history: 20 TP->UNFILLED (phantom wins), 23 EXPIRED->TP (missed touches), 4 EXPIRED->SL. DB backup: prisma/dev.db.bak-2026-09-10-before-fill.

Not ported to m5/roi-plan-readback.js (chart cross-check). Related: [[trade-journal]]

## Range-break fix (2026-09-10, evening)
- 19:30 +07 bar broke range low 77600 (low 76634). All signals (T-89SE, T-9LKS) SL'd. Root cause: range = rolling window extreme, so rangeLow followed price down and the RANGE fade kept buying "the bottom"; regime efficiency 0.1-0.32 (<0.4) because the 40-bar window held the prior chop; rangePos could go negative and still pass.
- Fix: lib/range.ts `rangeState(window, k)` - edges from bars before the last k, broken if any of the last k CLOSES is beyond (wicks do not count). plan.ts `rangeHeld` feeds entryOk (gateEntry), backtest filters, reasoning text; scripts/watch.ts suppresses the broken side's zone alert. Config `rangeBreakBars` (TRACE_RANGE_BREAK_BARS) default 6.
- Sweep (npm run tune -- range 30m): k=0 2.94x/worst yr 0.09%/DD 8% (2552 trades); k=6 2.84x/0.20%/7% (2130); k=12 2.83x/0.16%/11%; k=24 1.95x; k=48 1.51x. Modest risk improvement, not a cure; 2025 per-trade got worse (0.38->0.20).
- After a break the system stands aside; trading the breakout was not built.
- Fixtures T-AQ7Y/T-RU8X (post-break, must reject) and T-7SE9/T-JDDG (range intact, must not) in lib/plan.incident.test.ts.
- Stop-default question (minStopLevelAtr 0.5 vs 1 vs revert) still unanswered by the user.

## Dashboard trade-status banner (2026-09-10, night)
- lib/tradeStatus.ts decides STAND_ASIDE (latest rangeHeld=false) > POSITION_OPEN > TRADE (signal PENDING, unfilled, not taken) > NO_SIGNAL; app/page.tsx renders it above the board and lists only live signals (finished signals used to show as "recommendations" because only taken ones were filtered). "take it anyway" hidden during STAND_ASIDE.
- Snapshot columns rangeHeld/rangeEdge/rangeBrokeAt/rangeBreakClose (migration 20260910150000_add_range_break); null on older rows = unknown, not broken.
- Re-check time = breakAt + rangeBreakBars * timeframe (the breaking bar becomes established range when the k-th bar after it opens). First version had k+1; live data proved it: 19:30 break healed at the 22:30 bar.
- OPEN FLAW: entries below the range edge still pass (rangePos negative passes `rangePos <= maxRangePos`). T-MWAU (22:31, LONG limit 76479, sl 76169) sits under the new range low 76634 - fills only if the low breaks again. Proposed fix: fade entries must sit on the inside of rangeEdge.
