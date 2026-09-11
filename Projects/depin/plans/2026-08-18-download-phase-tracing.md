---
title: Download phase tracing - edge, SDK, worker, relay
date: 2026-08-18
project: depin
status: implemented
tags: [depin, download, metrics, monkit, observability, edge, worker, relay, go-sdk]
---

# Download phase tracing: edge, SDK, worker, relay

## Context

We cannot say where a download's time goes. Live measurements put the direct-SDK
ceiling near 8-9 MiB/s and the edge near 4.7-5.3 MiB/s, and the working
hypothesis in `edgeserver/config.go:36-42` is that a download is gated by how
long its slowest piece takes to **open**, not by bandwidth. That hypothesis is
currently unfalsifiable: `piece_open` is a single opaque number that fuses order
signing, libp2p dial (relay circuit vs direct), TLS handshake, gRPC stream open,
and the request frame send. There is no first-byte phase at all, no per-worker
attribution, and no way to follow one slow download from the edge into the
worker that served it.

The instrumentation that does exist is good and must be built on, not replaced:

- Edge emits `edge_download_*` request metrics and `edge_sdk_phase_*` per-phase
  durations (`edgeserver/metrics.go`).
- go-sdk has a per-transfer `metrics.Collector` with `Timer`/`Max`/`Reader`/
  `ConcurrentReader`/`Transformer` primitives
  (`../go-sdk/internal/metrics/metrics.go`) whose snapshot reaches the edge
  through `MetricsSink`.
- Worker emits `download_*` outcome counters plus two TTFB values
  (`worker/piecestore/endpoint.go`).
- Relay emits RTT percentiles, rejections, circuit lifetime, byte rates
  (`coord/relay/observer.go`, `coord/relay/tracer.go`).
- Everything exports through monkit -> `pkg/telemetry` remote_write and
  `pkg/debug` `/metrics`.

Intended outcome: for any window we can decompose end-to-end download latency
into named, non-overlapping phases; say which of them is relay-caused; name the
workers in the tail; and reconstruct a single slow request across edge and
worker logs. Both the edge path and the direct-SDK path are measured, and one
Grafana dashboard shows the phase waterfall for both.

## Decisions taken

- **monkit stays the instrumentation library.** No OpenTelemetry. `otel` in
  `go.mod` is indirect only.
- **No histograms.** `pkg/telemetry` drops monkit reservoir quantiles by
  default; `sum` and `count` are cumulative, so `rate(x{field="sum"})` gives
  seconds-of-phase per second of wall clock, which is exactly the waterfall
  view. `field="max"` covers the tail.
- **go-sdk is in scope** (user-confirmed). Most phases only exist there.
- **Always-on production metrics**, not a debug-only trace mode.
- **Cardinality is bounded by design**: phase and step names are fixed and
  small; only two metrics carry a `worker_id` label, and both are documented
  with an `AIOZ_METRICS_EXCLUDE_PREFIXES` escape hatch, following the precedent
  in `edgeserver/metrics.go:222-224`.

## Build order

WS1 and WS3-WS5 are independent. WS2 depends on WS1. WS6 depends on all.

---

### WS1 - go-sdk: split `piece_open` and close the timing blind spots

Files: `../go-sdk/internal/workerclient/download.go`,
`../go-sdk/internal/piecedownload/manager.go`,
`../go-sdk/internal/segmentdownload/segment.go`,
`../go-sdk/download.go`, `../go-sdk/internal/metrics/metrics.go`

1. **Widen the piece timer.** `PieceTimer` (`manager.go:64`) currently takes
   `(phase string, d time.Duration)`. Extend `workerclient.Client.GetPiece`
   (`internal/workerclient/download.go:22`) to report its internal steps through
   a new optional `StepTimer` callback, timed at the sites that already exist:
   - `piece_sign_order` around `orders.next(ctx, 0)` (`download.go:70`)
   - `piece_dial` around `c.dialer.DialNodeURL` (`download.go:96`), which is
     already bracketed by `dialStart` at `download.go:95` for the debug log
   - `piece_stream_open` around `pieceClient.Download(streamCtx)`
     (`download.go:110`)
   - `piece_request_send` around `stream.Send(...)` (`download.go:117`)
   Keep reporting `piece_open` as the sum, so existing dashboards and the
   `edge_sdk_phase_piece_open` series do not break.

2. **Add `piece_first_byte`.** Today `piece_open` stops once the request frame
   is sent, and nothing measures the wait for the worker's first chunk. Time the
   first `r.stream.Recv()` in `downloadStreamReader.Read`
   (`internal/workerclient/download.go:164`) and report it once per piece.
   Record it with `Collector.Max` as well as `Add`, since a segment is gated by
   the slowest piece (same rationale as `piece_slowest`,
   `metrics.go:108-123`).

3. **Add `admission_wait`.** `Manager.OpenReaders` blocks on the client-wide
   semaphore at `manager.go:180` (`internal/piecedownload/limiter.go:78-96`) and
   on the per-segment window at `manager.go:291-297`. A prior investigation
   attributed roughly 12.4s of a 13.45s latency increase to pre-piece queueing
   with no metric to prove it. Time both, tagged `scope=global|window`.

4. **Add `coord_resolve`.** `wallStart` and the `Collector` are created *after*
   the coordinator RPC (`download.go:490-491`, `download.go:715-716`), so ticket
   resolve is invisible to `MetricsSink`. Create the collector in the entry
   points (`DownloadFile` `download.go:221`, `DownloadByTicket` `download.go:356`,
   `ResolveTicket` `download.go:557`, `ResolveTicketRange` `download.go:575`,
   and the ranged pair) and pass it into `streamManifestOnce` /
   `streamManifestRangeOnce`. Time the `GetDownloadInfo*` calls at
   `download.go:230`, `:372`, `:584`, `:640` and the retry re-resolves at
   `download_retry.go:217`.

5. **Add `client_write`.** Time the destination writes in `streamSegments`
   (`download.go:419`) so "we produced bytes faster than the consumer took them"
   is separable from SDK work. For the edge this is the read-ahead queue; for a
   direct SDK caller it is the disk.

6. **Fix the two lying phases on the ranged path.** `erasure_decode` is
   reader-timed only on the ranged path (`segment.go:494`), and ranged `decrypt`
   is reader-timed rather than Transformer-timed (`segment.go:515`), so both
   fold network waiting into what claims to be CPU. This is the exact bug the
   whole-file path already fixed (`internal/metrics/metrics.go:34-50`). Move the
   ranged path onto `Collector.TransformReader` (`transform.go:68`) so it emits
   `decrypt` plus `pipeline_stall` with the same meaning as the whole-file path.

   Sanity check to run afterwards, from `edgeserver/metrics.go:244-246`: a
   phase claiming to be CPU must not exceed the process's `proc_stat`
   Utime+Stime over the same window.

**New phase names reaching `MetricsSink`** (and therefore
`edge_sdk_phase_*` automatically, via the loop at `edgeserver/metrics.go:247`):
`coord_resolve`, `admission_wait`, `piece_sign_order`, `piece_dial`,
`piece_stream_open`, `piece_request_send`, `piece_first_byte`, `client_write`,
plus `erasure_decode` and `pipeline_stall` now correct on both paths.

---

### WS2 - go-sdk: monkit export + per-worker and per-transport attribution

Files: new `../go-sdk/monkit_download.go` (root package, where
`var mon = monkit.Package()` already lives at `client.go:373`),
`../go-sdk/internal/piecedownload/manager.go`, `../go-sdk/options.go`

The `Collector` snapshot only escapes through `MetricsSink`, which only the edge
registers. A direct SDK download (`../go-sdk/cmd/sdk-benchmark`) produces no
metrics at all. Fix by having the SDK emit to `monkit.Default` itself, in
addition to calling the sink.

| Metric | Type | Tags | Cardinality |
|---|---|---|---|
| `sdk_download_phase_<phase>` | DurationVal | - | ~12 |
| `sdk_piece_open_duration` | DurationVal | `step`, `transport`, `cached` | 4 x 3 x 2 |
| `sdk_piece_first_byte_duration` | DurationVal | `transport` | 3 |
| `sdk_piece_duration` | DurationVal | `outcome` (ok/failed/abandoned) | 3 |
| `sdk_admission_wait_duration` | DurationVal | `scope` | 2 |
| `sdk_coord_resolve_duration` | DurationVal | `rpc` | ~4 |
| `sdk_piece_slowest_worker` | Counter | `worker_id` | 1 per worker |
| `sdk_worker_piece_open_duration` | DurationVal | `worker_id` | 1 per worker |

- `transport` = `relay|direct|unknown`. This is the single most valuable label
  in the plan: it directly answers whether relay hairpin is the cost. Source it
  from the connection's multiaddr, the same way `internal/p2pmonitor/observer.go`
  already classifies direct vs relayed peers.
- `cached` = whether `pool.Get` served from cache. The pool already measures
  this (`connection_from_cache` / `connection_dialed` /
  `connection_dial_duration`, `internal/grpcutil/pool/pool.go:242-256`); thread
  the outcome out of the dial so it can tag the piece.
- `sdk_piece_slowest_worker` increments only for the worker that was the slowest
  contributor to a segment. One series per worker (~150 today), and it is the
  direct answer to "who is the tail".
- `sdk_worker_piece_open_duration` is the per-worker open cost. Default on;
  document `AIOZ_METRICS_EXCLUDE_PREFIXES=sdk_worker_` to drop it if the fleet
  grows past what Prometheus should carry.

`edge_sdk_phase_*` stays as-is so nothing already deployed breaks; the new
`sdk_*` series are what the direct-SDK arm and any future embedder get.

---

### WS3 - Cross-component correlation: one download id, edge to worker

Files: `edgeserver/handler.go`, `../go-sdk/download.go` (new exported helper),
`../go-sdk/internal/workerclient/download.go`, `internal/grpcutil/log.go`,
`worker/piecestore/endpoint.go`

1. Edge mints a `download_id` per request in `serve` (`handler.go:76`), adds it
   to every log line for that request, and returns it as an
   `X-Download-Id` response header so a failing client can be traced.
2. New exported SDK helper `uplinksdk.ContextWithDownloadID(ctx, id)`. The SDK
   attaches it as gRPC metadata (`x-aioz-download-id`, plus `x-aioz-piece-num`
   on the piece stream) on the coordinator `GetDownloadInfo*` calls and on
   `pieceClient.Download` (`internal/workerclient/download.go:110`).
3. `AddGrpcLogContext` and `AddGrpcLogContextStream` (`internal/grpcutil/log.go:26`,
   `:78`) already mint a server-side `request_id`. Extend both to prefer an
   incoming `x-aioz-download-id` and to thread it into the handler context, so
   the worker's own `Download` logs and its success/failure summary defer
   (`worker/piecestore/endpoint.go:765-802`) carry it.

Result: one Loki query on `download_id` returns the edge request line plus every
worker line that served a piece for it.

**Stated limit, deliberately not solved:** the relay terminates an encrypted
libp2p circuit and cannot see gRPC metadata. Relay correlation stays
peer-id-and-time based. That is why WS5 adds aggregate relay signals rather than
per-request ones.

---

### WS4 - Worker: serve-path phase breakdown

File: `worker/piecestore/endpoint.go` (`Endpoint.Download`, `:681-972`),
plus `worker/peer.go`

All new metrics reuse the existing `actionTag` (`endpoint.go:720`) so they match
the current `download_*` family.

| Metric | Site |
|---|---|
| `download_verify_orderlimit_duration` | `verifyOrderLimit` `:728` |
| `download_verify_order_duration` | `verifyOrder` `:744` |
| `download_begin_save_order_duration` | `beginSaveOrder` `:842` |
| `download_reader_open_duration` | `pieceBackend.Reader` `:857` |
| `download_order_wait_duration` | `tracker.Await` `:908` |
| `download_disk_read_duration` | `io.ReadFull(reader, ...)` `:923` |
| `download_send_duration` | `stream.Send(...)` `:931` |

The last three are the point of this workstream. `tracker.Await` is the worker
blocking on the client's progressive order authorization; `stream.Send` is
network backpressure through the relay; `io.ReadFull` is hashstore disk. Today
all three are invisible and all three are indistinguishable inside
`download_success_duration_ns`.

Also:

- Export `e.liveRequest` (`endpoint.go:688`) as a `live_download_requests`
  gauge. It is currently an atomic that nothing reads out.
- Register the hashstore StatSource: `HashStoreBackend.Stats`
  (`worker/piecestore/backend.go:161-190`) builds a proper monkit series but
  there is no `mon.Chain(peer.Storage.HashStoreBackend)` anywhere, so it never
  reaches `/metrics` or the push client. Add the chain at the construction site
  (`worker/peer.go:458-474`), following the pattern at
  `internal/p2pmonitor/observer.go:61`.

---

### WS5 - Relay: the two missing signals

File: `coord/relay/tracer.go`, `coord/relay/observer.go`

Relay coverage is already good, so this is deliberately small:

- `relay_circuit_open_duration` DurationVal - time from CONNECT to circuit
  established, in the tracer alongside `relay_circuit_lifetime`
  (`tracer.go:128`).
- `relay_conn_rate_limited` Counter - connection rate limiting is configured
  (`internal/p2putil/resourcelimit.go:141`) and completely unmeasured, so a
  throttled dial today looks identical to a slow one.
- `relay_circuit_bytes` IntVal - per-circuit byte distribution, so "many small
  circuits" is separable from "few large ones".

---

### WS6 - Export both arms, and the dashboard

1. **Direct-SDK arm.** `../go-sdk/cmd/sdk-benchmark` has no metrics wiring.
   Extend it to aggregate the per-download phase snapshots and push
   `dl_bench_phase_seconds{phase=...}` to the Pushgateway alongside the existing
   `dl_bench_*` series, reusing the `testid`/`arm`/`conc`/`global_conc` grouping
   labels that `dev/bench/download-test.sh:120` already establishes. This keeps
   the sdk arm inside the harness that already exists rather than inventing a
   second one.

2. **Edge arm** needs no new export path: the new `edge_sdk_phase_*` names ride
   the existing `MetricsSink` loop, and `sdk_*` rides `monkit.Default` through
   `pkg/telemetry`.

3. **Edge-local gaps to close while here** (`edgeserver/metrics.go`,
   `edgeserver/handler.go`):
   - `edge_download_phase_resolve` and `edge_download_phase_transfer` - today the
     only edge spans are the two `time.Now()`/`time.Since` pairs at
     `handler.go:187`/`:242` and `handler.go:351`/`:387`, and ticket resolve at
     `handler.go:164` is inside neither.
   - `edge_download_ttfb` - first byte to the client.
   - `edge_readahead_wait` / `edge_readahead_drain` / `edge_readahead_queue_bytes`
     - the new `edgeserver/readahead.go` emits nothing, so we cannot tell whether
     the buffer is helping or whether a slow client is the bottleneck.
   - `edge_head_requests` / `edge_head_duration` - `serveHead`
     (`handler.go:402-415`) emits no metric at all.

4. **Dashboard**: new `monitoring/grafana/dashboards/download/download-phases.json`
   plus a `download` -> "Download" provider in
   `monitoring/grafana/provisioning/dashboards/dashboards.yml`. Panels:
   - **Phase waterfall** (stacked): `rate(edge_sdk_phase_*{field="sum"}[$__rate_interval])`,
     seconds of phase per second of wall clock. One row per arm (edge / sdk).
   - **piece_open sub-phase breakdown** (stacked): sign / dial / stream_open /
     request_send / first_byte.
   - **Relay vs direct**: `sdk_piece_open_duration{transport=...}` and
     `sdk_piece_first_byte_duration{transport=...}` side by side. This panel is
     the payoff of the whole plan.
   - **Pool effectiveness**: `cached=true` vs `false` open cost.
   - **Tail workers**: top-10 table on `topk(10, rate(sdk_piece_slowest_worker[$__rate_interval]))`
     joined with `sdk_worker_piece_open_duration{field="ravg"}`.
   - **Admission**: `sdk_admission_wait_duration` by scope, next to
     `edge_download_rejected`.
   - **Worker serve path** (stacked): the seven new `download_*_duration` series.
   - **Relay**: reuse the existing `relay_*` panels plus the two new ones.
   - **Budget reconciliation**: `edge_download_duration` minus the sum of the
     attributed phases, so unexplained time is a visible number rather than an
     assumption.

5. Update `monitoring/README.md`. It currently stops at section 9 (Edge) and
   still describes two dashboard providers when `dashboards.yml` defines six.

---

## Constraints and traps

- **go-sdk is a separate repo behind `replace aioz-depin/go-sdk => ../go-sdk`
  (`go.mod:415`).** CI has no `../go-sdk`, so depin will not build in CI until
  the go-sdk change is pushed and the pin at `go.mod:64` is updated. Land go-sdk
  first, push, re-pin, then land the depin side. This has bitten this repo
  before.
- **Do not re-introduce the CPU-accounting bug.** Reader-timed phases include
  waiting. Any phase presented as CPU must be Transformer-timed
  (`../go-sdk/internal/metrics/transform.go:56-86`), and must pass the
  `proc_stat` check above.
- **`piece_download` is a union, not a sum** (`ConcurrentReader`,
  `metrics.go:147`). Do not stack it against per-piece phases without saying so
  in the panel description.
- **Phases must not double count.** `piece_open` stays the sum of its four new
  steps; assert this in a unit test rather than trusting it.
- The auto-formatter in this workspace re-wraps whole Go files and has mangled
  the go-sdk `common` import before. Verify loop bodies and imports after edits.

## Verification

1. **Unit**: extend `../go-sdk/internal/metrics` tests for the new phases;
   assert `piece_open == sign+dial+stream_open+request_send`; assert the ranged
   path now emits `pipeline_stall` and a Transformer-timed `decrypt`. Extend
   `edgeserver/metrics_test.go` (which already asserts phase counts at
   `metrics_test.go:145-150`) for the new names. Worker: table test that every
   new `download_*_duration` is observed exactly once per served piece, using
   the scoped-registry pattern (`withTestMon`) this repo already uses for monkit
   producers.
2. **e2e (testplanet)**: a download through the edge asserts that a scoped
   monkit registry contains every new series with a non-zero count, and that
   the same `download_id` appears in both the edge and worker log capture
   (zaptest/observer). This is the real guard on WS3.
3. **Live**: run `./dev/bench/download-test.sh -a both -c 4` against the deployed
   stack (concurrency 4, not 8, per the production-contention finding) and
   confirm the new dashboard renders both arms. Then answer the original
   question with data: does `piece_dial` + `piece_first_byte` under
   `transport="relay"` dominate, and is `admission_wait` material?
4. **Cardinality check**: after one live run, count series with
   `count({__name__=~"sdk_.*"})` and `count by (__name__)({__name__=~"sdk_worker_.*"})`
   and record the numbers in `monitoring/README.md`.
5. `go build ./...` in both repos after every merge; the LSP is unreliable on the
   symlinked go-sdk here.
