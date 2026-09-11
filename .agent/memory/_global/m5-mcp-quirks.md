---
tags: [memory, m5, mcp, trading]
created: 2026-09-04
---

# M5 MCP quirks

The M5 charting app (app.mmt.gg) exposes an MCP server used for chart/script automation.

## Transport
- Plain streamable HTTP: `https://app.mmt.gg/api/mcp`, `Authorization: Bearer m5_mcp_...`.
- Config lives in `~/.claude.json` under `projects["/home/tuan/personal/essay-manager"].mcpServers.m5`.
- Stateless: `initialize` returns 200 JSON with no `mcp-session-id`, so **any process can call it** -
  cron scripts do not need a Claude session. Use `@modelcontextprotocol/sdk` +
  `StreamableHTTPClientTransport`.

## No data read
There is no tool that returns market data (candles, order book, positions, plan values). Tools are
write/control only, plus `m5_get_status`, `script_list/get/diagnostics`.

**Workaround that works:** a script can `throw new Error(<computed values>)`; the message comes back
verbatim in `script_get_diagnostics` as `data.diagnostics[0].detail`
(`"create_runtime: PLAN side=LONG entry=... "`). That is the only readback channel.
The error appears per freshly created layer, so the caller does `chart_add_script` then polls
diagnostics; a layer already stuck in an error state can stay quiet.

## Broken builtins
- `math.roundToTick()` is documented but **crashes the script runtime**: any script calling it fails
  with `failed to create runtime` and never loads (no visible error on the chart). Round manually:
  `math.round(v / tickSize) * tickSize`.
- `script_create`/`script_update_code` return `diagnostics.status`: `"ok"` means it compiled,
  `"failed to create runtime"` means the code is broken, `"unavailable"`/`"timed_out"` means no live
  browser session could verify it. Bisecting with tiny probe scripts is the fastest way to find which
  builtin kills a script.

## Browser dependency
All data originates in the user's open M5 browser tab. No tab connected -> `m5_get_status` returns no
sessions and nothing can be captured. Layers are lost on reload unless the workspace is saved.

Related: [[trace-trade-journal]]
