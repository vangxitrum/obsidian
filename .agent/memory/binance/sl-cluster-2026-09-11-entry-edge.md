---
type: decision
tags: [binance, entries, signals, dashboard]
created: 2026-09-11
agent: main
---

# Entry-edge rule (reverted), stop level 1 ATR, "gates pass but no trade" (2026-09-11)

- User chose `minStopLevelAtr` = 1 (config default; backtest 3.09x vs old 3.16x).
- Entry-edge rule TRIED AND REVERTED: requiring fade entries inside the window range (rangePos >= 0) rejected T-MWAU/T-8RY7-style limits (entry offset 0.5 ATR pushes the limit under a swing low that is also the range low), but backtest (npm run tune -- range 30m, k=6, level 1) fell 3.00x -> 1.21x, worst year 0.29% -> -0.23%, DD 7% -> 14%. Wick-under-the-low fills are where the fade's edge is; real breaks are handled by rangeHeld. User chose revert. lib/plan.incident.test.ts now asserts these entries are ALLOWED.
- "All gates pass - see the recommendation below" with nothing below: lib/signal.ts compares each reading to the PREVIOUS reading, and with the watcher capturing every minute a chain of passing readings stays "the same setup was already raised" (origin T-U38S 05:29, filled 06:08). User chose to explain, not re-raise: lib/tradeStatus.ts `gatesNote` + a gates-pass NO_SIGNAL branch naming the open-on-paper signal or the suppression reason; signal rules unchanged.
- Backtest numbers in [[sl-cluster-2026-09-10]] follow-up; see config.ts comments.
