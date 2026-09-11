---
tags: [memory, trace, trading, nextjs]
created: 2026-09-04
---

# binance - trade plan journal

Renamed from `trace` to `binance` on 2026-09-07 (directory, package name and Go module
only - the `TRACE_*` env vars were deliberately kept).

`/home/tuan/personal/binance` - Next.js 16 + Prisma 7 + SQLite app that records M5 trade-plan
snapshots. Dev server on **port 3011** (3010 was already taken on this machine).

## Why it exists
The M5 script `ROI Entry Planner` (id `e882d94f-2be0-445c-8db9-318dc7002ba3`) computes a leveraged
BTC entry plan; `ROI Plan Readback` (id `7580779b-04ff-436a-9f61-bbbd56617523`) returns the same
numbers through the throw-into-diagnostics trick (see [[m5-mcp-quirks]]). The app captures that every
4h via cron and keeps plan + reasoning + outcome.

## Key decisions
- **"15-20%" means ROI on margin, not price move.** At 5x that is a 3-4% price move, hold measured
  in days. Target price move = `targetRoi / leverage`.
- Three gates decide `TRADEABLE`: loss ROI <= max (default 10%), liq distance >= 3x stop distance,
  R:R >= 3.
- Outcome resolution uses `gapHigh`/`gapLow` (extremes since the previous capture), not closes, so a
  touch between captures is still seen. Both touched in one window resolves as **SL** - the order is
  unknowable and assuming the win would flatter the record.
- Reasoning text is rule-based, not an LLM call, so cron stays deterministic and offline.
- `CaptureRun` rows record skips ("no browser session"), so a data gap is distinguishable from cron
  never firing.

## Operational
- Cron: `0 */4 * * *` running `scripts/capture.sh`; **requires an M5 browser tab open**.
- `.env` (chmod 600) holds `M5_MCP_TOKEN`; never commit it.
- Prisma 7 adapter export is `PrismaBetterSqlite3` (lowercase "qlite"), not `PrismaBetterSQLite3`.

Related: [[m5-mcp-quirks]], [[live-order-sidecar]]
