# Edge-server download-speed monitoring

Date: 2026-07-13
Status: design approved (pending spec review)
Repos touched: `depin` (this repo) and `../go-sdk` (sibling, `replace aioz-depin/go-sdk => ../go-sdk`)

## Goal

Make the edge server's download speed visible in the existing monitoring stack
(Grafana). Two distinct speeds are surfaced:

1. **Served throughput** - bytes streamed to the end user divided by request
   wall time. This is what the end user actually experiences from the edge.
2. **SDK fetch throughput** - the internal speed at which the go-sdk download
   engine pulls pieces from workers, Reed-Solomon-decodes, and decrypts, plus a
   per-phase time breakdown. Reflects worker-fetch health, not end-user speed.

Both flow through the same path the coordinator roles already use:
`monkit.Default` -> `pkg/telemetry` remote_write push -> Prometheus -> Grafana.

## Background (current state)

- `edgeserver` serves `GET /download?ticket=...` and streams reconstructed,
  decrypted file content via `client.DownloadByTicket(ctx, raw, w, ...)`
  (`edgeserver/handler.go`). `w` is the HTTP `http.ResponseWriter`.
- `edgeserver` already registers `monkit.Default` on a private localhost debug
  endpoint (`edgeserver/server.go`), but has **no** `pkg/telemetry` remote_write
  push client, so nothing reaches the monitoring VPS today. The monitoring stack
  currently receives from `coord` roles only.
- The go-sdk already computes the numbers we need for SDK fetch speed:
  `DownloadManifest` (`../go-sdk/download.go`) tracks `totalBytes` and `wall`
  and holds a `*metrics.Collector` (`../go-sdk/internal/metrics/metrics.go`)
  with a per-phase `Snapshot()`. Today it only feeds `LogSummary` (a single log
  line); it is not exposed to callers.
- `coord` establishes the reusable push pattern: `setupTelemetry`
  (`coord/peer.go`) builds `telemetry.NewClient` only when `Metrics.URL` is set
  (off by default) and registers it as a lifecycle item. `coord.Config` embeds
  `Metrics telemetry.Config` (`coord/config.go`), which `process.Bind` turns
  into `--metrics.*` / `AIOZ_METRICS_*` flags.

## Design

### A. Edge-side served-throughput instrumentation (`depin/edgeserver`)

New file `edgeserver/metrics.go` declaring `var mon = monkit.Package()` and the
edge metrics. Recorded once per `/download` request from `handleDownload`:

| Measurement | monkit type | Meaning |
|---|---|---|
| `edge_download_bytes` | `IntVal` | bytes streamed to the client |
| `edge_download_duration` | `DurationVal` | request wall time |
| `edge_download_speed_bps` | `IntVal` | served bytes / wall seconds (headline) |
| `edge_download_success` | `Counter` | successful downloads |
| `edge_download_error` | `Counter` | failed downloads |

`handleDownload` wraps `w` in a byte-counting `http.ResponseWriter` and starts a
timer before calling the SDK. On return it computes bytes/wall, records the four
value metrics, and increments success or error. `speed_bps` is only recorded
when `wall > 0` and `bytes > 0` (avoid divide-by-zero / meaningless zero-byte
samples).

The counting wrapper forwards `Header()`, `Write()`, and `WriteHeader()` and
counts bytes from `Write`. It must also pass through the `WriteHeader` status so
the existing metadata-header behavior is unchanged.

### B. SDK stats sink (`../go-sdk`) - already exists, no change needed

The go-sdk already exposes exactly the hook we need: a `WithMetricsSink`
`ClientOption` (`../go-sdk/options.go`) whose callback

```go
type MetricsSink func(op string, snapshot map[string]time.Duration, bytes int64, wall time.Duration)
```

is invoked synchronously at the end of every transfer. `DownloadManifest`
(`../go-sdk/download.go`) already calls `c.metricsSink("download",
coll.Snapshot(), totalBytes, wall)`. So the edge only has to register a sink at
client-construction time; **no go-sdk edit is required.** (`op` is `"download"`
or `"upload"`; the edge guards on `op == "download"`.)

### C. Edge SDK-fetch metrics (`depin/edgeserver`)

`handleDownload` registers a stats sink via `WithStatsSink`. On invocation it
records:

| Measurement | monkit type | Meaning |
|---|---|---|
| `edge_sdk_fetch_speed_bps` | `IntVal` | SDK bytes / SDK wall seconds |
| `edge_sdk_fetch_duration` | `DurationVal` | SDK total wall time |
| `edge_sdk_phase_<name>` | `DurationVal` | per-phase net duration, one measurement per phase key |

Phase measurement names are built at runtime:
`mon.DurationVal("edge_sdk_phase_" + sanitize(phase)).Observe(d)`, where
`sanitize` lowercases and replaces any non `[a-z0-9_]` (e.g. the `.` in
`coord.GetDownloadManifest`) with `_`. monkit encodes the phase in the
measurement name because monkit has no per-observation label primitive; the
telemetry collector sanitizes measurement names on the way out regardless.
Phase keys come from the SDK `Snapshot()` map (e.g. `piece_transfer`,
`erasure_decode`, `decrypt`, `coord.*` RPC timers).

### D. Telemetry push (`depin/edgeserver`)

- Add `Metrics telemetry.Config` to `edgeserver.Config` (`edgeserver/config.go`).
  `process.Bind` (already wired in `cmd/edgeserver/main.go`) then exposes the
  same `--metrics.*` / `AIOZ_METRICS_*` flags coord has, off by default.
- Add a `setupTelemetry`-equivalent to `edgeserver.New` / `Server.Run`: build
  `telemetry.NewClient(logger, cfg.Metrics, monkit.Default)` **only when**
  `cfg.Metrics.URL != ""`, and add `client.Run` / `client.Close` to the
  existing `errgroup` in `Server.Run` (register a lifecycle goroutine mirroring
  the `debugSrv` branch). Uses `monkit.Default` - the same registry the debug
  endpoint already exposes - so no double registration.

**Labels:** per decision, `telemetry.Config` defaults are kept as-is (no forced
`app`/`role` override in edge code). Because the edge's measurements are all
prefixed `edge_*`, the dashboard keys on measurement name and does not depend on
the `app` label. Operators SHOULD set `AIOZ_METRICS_APP=edgeserver` (and
optionally `AIOZ_METRICS_ROLE=<fleet/region>`) so edge series are grouped
distinctly from coord's `app=coord`; this is documented, not enforced.

### E. Grafana dashboard (`depin/monitoring`)

New `monitoring/grafana/dashboards/edge-downloads.json`, auto-provisioned. Add a
second provider entry to
`monitoring/grafana/provisioning/dashboards/dashboards.yml` so edge dashboards
load into a distinct **Edge** folder (the existing provider loads coord
dashboards into **Coord**). Every panel references `${DS_PROMETHEUS}` like the
existing dashboards.

Panels (all keyed on `edge_*` measurement names, `app` label optional). Note
the monkit->Prometheus convention: each aggregate is a `field` **label** on the
base measurement name, not a name suffix (see `monitoring/README.md` section 7):
- **Served download speed** - `edge_download_speed_bps{field="ravg"}`
- **SDK fetch speed** - `edge_sdk_fetch_speed_bps{field="ravg"}`
- **Bytes served rate** - `rate(edge_download_bytes{field="sum"}[5m])`
- **Download request rate** - `rate(edge_download_success{field="value"}[5m])`
  and `rate(edge_download_error{field="value"}[5m])`
- **Request duration** - `edge_download_duration{field="ravg"}`
- **SDK phase breakdown** -
  `{__name__=~"edge_sdk_phase_.*", field="ravg"}` (one series per phase)

### F. Documentation (`depin/monitoring/README.md`)

Add an edge-server subsection: what the edge pushes, the required/recommended
env vars (`AIOZ_METRICS_URL`, `AIOZ_METRICS_APP=edgeserver`,
`AIOZ_METRICS_ROLE`, bearer token - same receiver as coord), the metric table
from A/C, and the example queries from E. Note the edge shares the exact same
receiver, token, and interval semantics as coord roles.

## Testing

- **Unit (edge, `edgeserver`):**
  - Counting `http.ResponseWriter` counts exactly the bytes written and passes
    header/status through.
  - `handleDownload` with a stub client (via `NewFromClient`) records the edge
    metrics: assert `edge_download_*` moved by walking a monkit registry
    snapshot; assert error path increments `edge_download_error`.
  - Telemetry off by default: mirror `coord/telemetry_test.go` - with
    `Metrics.URL == ""`, no client is constructed and no lifecycle item added.
- **Unit (SDK, `../go-sdk`):** `WithStatsSink` fires once with `Bytes`/`Wall`
  matching the streamed total and a non-nil `Phases` map for a multi-phase
  download.
- **E2E verify (preferred):** run `edgeserver` with `AIOZ_METRICS_URL` pointed
  at a local Prometheus (the README's git-ignored `docker-compose.override.yml`
  that publishes `127.0.0.1:9090`), perform a real ticketed download, then
  `curl` Prometheus for `edge_download_speed_bps` and `edge_sdk_fetch_speed_bps`
  and confirm non-empty results; open the provisioned **Edge** dashboard and
  confirm the panels render.

## Out of scope

- Upload path, S3 API, and any coord-side change.
- Changing the monitoring stack's push-only model (no Prometheus scrape of edge
  debug endpoints).
- Forcing/renaming metric labels in edge code (decision: keep telemetry.Config
  defaults; document env vars instead).

## Open items

None blocking. Phase-breakdown cardinality is bounded by the fixed set of SDK
phase names; if it ever grows, `AIOZ_METRICS_EXCLUDE_PREFIXES=edge_sdk_phase_`
already lets an operator drop it without code changes.
