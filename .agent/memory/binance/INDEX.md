# binance — memory index

Trade plan journal capturing M5 chart plans on a schedule, and the Go sidecar that places the
resulting order on Binance. Renamed from `trace` on 2026-09-07.

- [[trade-journal]] — what this app is, the ROI-on-margin decision, gates, outcome resolution, cron + env setup.
- [[live-order-sidecar]] — the Go binary in `order/` that places real futures orders, its four guardrails, and why it is a separate process.
- [[sl-cluster-2026-09-10]] — 09-10 SL clusters: stop cause, fill-aware 1m resolver (UNFILLED), range-break guard (rangeBreakBars=6), backtest verdicts.
- [[sl-cluster-2026-09-11-entry-edge]] — stop level 1 ATR chosen; "entry inside range" rule tried and reverted (3.00x -> 1.21x); why "gates pass" can still mean no signal.
