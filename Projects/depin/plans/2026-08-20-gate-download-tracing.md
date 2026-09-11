# Gate detailed download tracing behind a config flag

## Context

The `feat/improve-download-speed` work added end-to-end download phase tracing
across `edgeserver`, `worker`, `internal/p2putil`, `internal/grpcutil` and the
`go-sdk` (depin commit `3896583` plus uncommitted work in both repos; in go-sdk
essentially the whole feature is still uncommitted). It answered the questions it
was built for - the relay's per-circuit `BufferSize` was the throughput ceiling,
and the edge read-ahead buffer was inert - but it is permanently on and it is not
cheap:

- ~30 extra monkit series, several of them per-piece with a `worker_id` tag
  (`sdk_worker_piece_duration`) - one series per worker in the fleet.
- `piece_recv_gap` alone was **595 observations per download**
  (`download-stats.md` §5); a 30 MiB download opens ~42 piece streams and each
  carries 6 `time.Now()` pairs.
- A `crypto/rand` download id + response header per HTTP request, and gRPC
  metadata on the coord resolve and every piece stream.
- An `rcmgr.TraceReporter` on libp2p's resource-manager hot path.

Goal: keep every bit of it, but only run it when an operator asks for it. Off by
default, one flag per component, read once at startup.

**Decisions (confirmed with the user):**

| Question | Decision |
|---|---|
| What is gated | **All** tracing added by this work - phase timings, `X-Download-Id` correlation, `p2p_resource_blocked`, the `HashStoreBackend.Stats` chain. Only metrics that predate the branch stay unconditional. |
| Default | **Off.** |
| Granularity | **One flag per component**: `edgeserver.Config.DetailedDownloadMetrics`, `piecestore.Config.DetailedMetrics`, `uplinksdk.WithDetailedMetrics(bool)`. The edge passes its own value into the SDK. |
| Toggle | **Startup-only** cfgstruct config. No runtime endpoint. |

## Guiding principle

The go-sdk already has four seams that are documented as "nil turns this off":
`*metrics.Collector` (nil is a total no-op, `internal/metrics/metrics.go:56-57`),
`workerclient.Config.StepTimer` (`internal/workerclient/client.go:34-35`),
`pool.WithStreamTimer` (returns ctx unchanged on nil,
`internal/grpcutil/pool/streamtimer.go:38-40`), and `piecedownload.WithTimer`
("without it nothing is recorded", `internal/piecedownload/manager.go:97`).

**Do not invent a parallel flag beside them.** Gate on the seam that already
exists, and extend that same condition to cover the `mon.DurationVal(...)` half
that currently fires unconditionally in the same functions. One new flag enters
each component at the top and turns into "no collector / no timer" underneath.

---

## 1. go-sdk (`/home/tuan/work/depin-workspace/go-sdk`)

### 1a. The option

`options.go` - add `detailedMetrics bool` to `options` (`:25-47`) and:

```go
// WithDetailedMetrics enables per-piece phase tracing (piece_sign_order,
// piece_conn_acquire, piece_stream_open, piece_first_byte, piece_recv_gap, ...).
// Off by default: a 30 MiB download opens ~42 piece streams and the detailed
// path costs several time.Now() pairs and a monkit observation per stream, plus
// a per-worker series. Coarse per-download metrics (sdk_download_duration /
// _bytes / _speed_bps) are always recorded.
func WithDetailedMetrics(enabled bool) ClientOption {
	return func(o *options) { o.detailedMetrics = enabled }
}
```

Positive sense with `false` as the zero value keeps
`options_test.go:111-113` (`TestClientOptions_NoneLeavesEverythingZero`) green.

`client.go` - add `detailedMetrics bool` to `Client` (beside `metricsSink`,
`:49`) and copy it in `newClient` (`:224`). **Update
`client_internal_test.go:19-56` (`TestNewClient_CopiesAllOptionsFields`)** - that
test exists precisely to catch a forgotten field.

### 1b. Threading it down (`download.go`)

In `streamManifestOnce` (`:480-497`) and `streamManifestRangeOnce`
(`:721-738`), replace the unconditional collector with a conditional one and
propagate:

```go
var coll *metrics.Collector
if c.detailedMetrics {
	coll = metrics.New()
}
wc := workerclient.NewWithConfig(c.dialer, c.logger, workerclient.Config{
	StepTimer: coll.StepTimerOrNil(),   // nil when coll == nil
	Transport: c.transport,
	Detailed:  c.detailedMetrics,
})
```

`StepTimer` must be a **nil func value**, not `coll.Observe` on a nil collector -
a method value off a nil pointer is non-nil, so `c.config.StepTimer != nil`
would stay true and the monkit half would keep firing. Either add a tiny helper
or write `if coll != nil { cfg.StepTimer = coll.Observe }`.

A nil `coll` then transitively silences `client_write` (`:497`, `:738`),
`piece_download`, `decrypt`, `pipeline_stall` (`internal/segmentdownload/segment.go:114,171,234,420,463,507`),
`piece_open`, `piece_slowest`, `admission_wait`
(`internal/piecedownload/manager.go`) - every one of those goes through a
nil-guarded `Collector` method.

At the terminal emission (`download.go:539-545`, `:786-792`) leave
`recordDownloadTransfer` and the `metricsSink` call in place. `coll.Snapshot()`
on nil returns nil, so the `sdk_download_phase_*` fan-out
(`monkit_download.go:99-101`) becomes an empty loop while
`sdk_download_duration` / `_bytes` / `_speed_bps` keep working. **The sink still
fires with an empty snapshot**, which is what keeps the edge's pre-existing
`edge_sdk_fetch_duration` / `edge_sdk_fetch_speed_bps` alive.

Gate `recordCoordResolve` at its two call sites (`download.go:140`, `:150`) on
`c.detailedMetrics` - `sdk_coord_resolve_duration` is new.

### 1c. Making the monkit half share the gate

These currently fire regardless of any timer. Each gets the nearest existing
condition rather than a new flag:

| site | metric | gate on |
|---|---|---|
| `internal/workerclient/steps.go:75-83` | `sdk_piece_open_duration{step,transport}` | `c.config.Detailed` |
| `internal/workerclient/download.go:189-191` | `sdk_worker_piece_open_duration{worker_id}` | `c.config.Detailed` |
| `internal/workerclient/download.go:307-317` | `sdk_piece_first_byte_duration{transport}` | `c.config.Detailed` |
| `internal/workerclient/download.go:321-330` | `sdk_piece_recv_gap_duration{transport}` | `c.config.Detailed` |
| `internal/piecedownload/manager.go:85-87` | `sdk_admission_wait_duration{stage}` | `m.timer != nil` |
| `internal/piecedownload/manager.go:496-498` | `sdk_piece_duration{outcome}` | `m.timer != nil` |
| `internal/piecedownload/manager.go:504-506` | `sdk_worker_piece_duration{worker_id}` | `m.timer != nil` |

Add `Detailed bool` to `workerclient.Config` (`internal/workerclient/client.go:28-42`),
documented as "also gates the monkit half" so the existing comment at `:34-35`
stops being a half-truth.

Skip the timing probes themselves, not just the emission - hoist the
`time.Now()` pairs in `GetPiece` (`download.go:116-188`) and
`downloadStreamReader.Read` (`:254`, `:264`) behind the same condition.

`internal/grpcutil/pool` needs no gate: `WithStreamTimer` is only called from
`workerclient/download.go:133-143`, so not calling it when detail is off leaves
`observeStep` a no-op by construction.

### 1d. Download-id correlation

`internal/downloadid/` (new package) + wherever `x-aioz-download-id` /
`x-aioz-piece-num` are attached to outgoing gRPC metadata: attach only when
`c.detailedMetrics`. The worker's `correlate()` reads whatever arrives, so
suppressing the producer is sufficient - no depin change needed for this half.

### 1e. `p2p_resource_blocked` and the p2pmonitor gauges

`internal/p2putil/resourcelimit.go` - add to `ResourceLimitConfig`:

```go
TraceBlocked bool `help:"count libp2p resource-manager refusals as p2p_resource_blocked{kind,scope}" default:"false"`
```

and make the `rcmgr.WithTraceReporter(blockedReporter{})` line conditional.
Mirror the identical change in depin's `internal/p2putil/resourcelimit.go:90`.

For the new gauges added to `internal/p2pmonitor/observer.go:215-217`
(`p2p_direct_peers`, `p2p_relayed_only_peers`, `p2p_direct_upgrades_total`):
gate **only what `git diff` added** to that file. The observer object itself
must keep running - `Client.transport` (`monkit_download.go:126-128`) depends on
it, and it predates this branch.

### 1f. `cmd/sdk-benchmark`

Pass `uplinksdk.WithDetailedMetrics(true)` unconditionally
(`cmd/sdk-benchmark/main.go:522` area). Measuring is the tool's entire purpose,
and `dev/bench/download-test.sh` now drives it with `-phase-out`. Keep
`cmd/sdk-benchmark/phases_test.go` green.

---

## 2. depin edge (`edgeserver/`)

### 2a. The flag

`edgeserver/config.go`, on the top-level `Config` (`:135-187`), matching the
package's `default` / `user` / `help` tag order:

```go
DetailedDownloadMetrics bool `default:"false" user:"true" help:"record per-download phase tracing (edge_resolve/transfer/ttfb, edge_readahead_*, edge_sdk_phase_*, X-Download-Id correlation); off by default because it costs several timers and a monkit series per piece"`
```

### 2b. Thread it - do NOT use a package-level global

`edgeserver/metrics.go:19` is a package `mon` and every `record*` is a
package-level function with no `Config` in scope, so a global `bool` is the
tempting shortcut. **Reject it**: testplanet constructs several edge servers in
one process from literal `edgeserver.Config{}` values
(`internal/testplanet/edge_range_test.go:37`, `edge_cache_test.go:41`,
`edgeserver_metadata_test.go:44`, `download_phases_test.go:62`), so a global
would leak one test's setting into another.

Instead pass `s.cfg.DetailedDownloadMetrics` at each site:

- `handler.go:173-176, 225-227, 259-262, 317-318, 349, 390-394, 415-418, 442-453` -
  guard `recordResolve` / `recordTransfer` / `recordTTFB` / `recordHead` and the
  `time.Now()` stamps that feed them. Simplest shape: give each `record*` a
  leading `detailed bool` parameter, or make them methods on `*Server`.
- `handler.go:89-91` - only mint the download id, set `X-Download-Id`, and call
  `uplinksdk.ContextWithDownloadID` when the flag is on. `reqLogger`
  (`downloadid.go:39-44`) then degrades to the bare logger on its own.
- `readahead.go` - `newReadAheadWriter(dst, budget)` (`:80`) gains a `detailed
  bool`. **Keep the buffering itself unconditional** - it is a throughput fix,
  not tracing. Gate only `blockedNs`/`queued`/`peak` accounting (`:158-180`) and
  the `recordReadAhead` call in `close()` (`:195-201`).
- `metrics.go:209-257` - turn `recordSDKTransfer` into a constructor
  `sdkTransferSink(detailed bool) uplinksdk.MetricsSink`, so
  `server.go:97` becomes
  `uplinksdk.WithMetricsSink(sdkTransferSink(cfg.DetailedDownloadMetrics))`.
  Keep `edge_sdk_fetch_duration` / `edge_sdk_fetch_speed_bps` unconditional
  (pre-existing); gate only the `edge_sdk_phase_*` loop at `:255`.
  `metrics_test.go:156-184` calls the sink directly, so it needs the new
  constructor.
- `server.go:84-113` - add
  `uplinksdk.WithDetailedMetrics(cfg.DetailedDownloadMetrics)` to the option
  list.
- `config.go:216` `sdkResourceLimits` - set
  `TraceBlocked: config.DetailedDownloadMetrics` on the returned
  `uplinksdk.ResourceLimitConfig`.

`NewFromClientConfig` (`server.go:184-197`) shares a caller-built SDK client and
cannot set SDK options - it already cannot register a `MetricsSink`. Its
edge-side flag still works; document that its SDK half follows the caller's
client.

---

## 3. depin worker (`worker/`)

### 3a. The flag

`worker/piecestore/service.go` `Config` (`:29-59`), matching that file's
`help`-first tag order:

```go
// DetailedMetrics turns on the download serve-path phase breakdown
// (download_verify_orderlimit/verify_order/begin_save_order/reader_open/
// order_wait/disk_read/send durations, the live-request gauge, and the
// download_id log field). Off by default: the send accounting runs inside the
// chunk loop, so it costs a time.Now() pair per Send on every piece served.
DetailedMetrics bool `help:"record the download serve-path phase breakdown and piecestore_live_requests gauge" default:"false"`
```

Reachable as `--worker.storage.detailed-metrics`. It lands on the `Endpoint`
(`endpoint.go:76-93`), which is the only place a `piecestore.Config` value is
actually stored - **`Service.config` is never set** (`NewPieceStoreService`
ignores it, `service.go:96-127`), so never read the flag off `svc`.

### 3b. Sites

`worker/piecestore/endpoint.go`:
- Add a small helper so the hot path stays readable and pays one branch instead
  of a `time.Now()` pair:
  ```go
  // phaseTimer records the detailed download phases when the endpoint is
  // configured for them. The zero value is off and every method is a no-op.
  type phaseTimer struct{ on bool }

  func (p phaseTimer) start() time.Time { if !p.on { return time.Time{} }; return time.Now() }
  func (p phaseTimer) observe(name string, start time.Time, tags ...monkit.SeriesTag) {
  	if !p.on { return }
  	mon.DurationVal(name, tags...).Observe(time.Since(start))
  }
  ```
- Apply to `:783`, `:797`, `:907`, `:931` (verify-orderlimit / verify-order /
  begin-save-order / reader-open). **Keep the control-flow restructure** the diff
  introduced (`verifyLimitErr := ...` then a separate `if`) - reverting to the
  inline `if err := ...` form would make the timing impossible to place and is
  churn for no gain.
- Guard the accumulator `defer` (`:977-999`) and the three probe pairs at
  `:1013/:1017`, `:1030/:1032`, `:1040/:1047-1052`.
- Guard the `download_id` log field (`:863-868`).
- `Endpoint.Stats` (`:95-106`): keep the method, but don't chain it when off.
  Leave the `liveDownload` atomic increments (`:731-732`) unconditional - an
  `atomic.AddInt32` is far cheaper than a branch plus a config read, and nothing
  reads it when unchained.

`worker/peer.go`:
- Guard `chainHashStoreStatsOnce` (`:476-481`) and `chainEndpointStatsOnce`
  (`:590-592`) on `config.Storage.DetailedMetrics`. Keep the `sync.Once`
  wrappers - the duplicate-series hazard they exist for is unchanged.
- `worker/server/server.go:57` `ResourceLimits` picks up the new
  `p2putil.ResourceLimitConfig.TraceBlocked` field automatically as
  `--worker.server.resource-limits.trace-blocked`.

---

## 4. One deliberate exception, stated plainly

`p2p_resource_blocked` gets its **own** `resource-limits.trace-blocked` flag
rather than hanging off the per-component download flag. Reason: it is a libp2p
infrastructure counter, not a download phase, and the component where it matters
most - the **relay** (`coord/relay/config.go:46`) - has no download flag to hang
it on. Making it a `ResourceLimitConfig` field gives every role (worker, edge,
coord, relay) the same knob for free.

For the two components the user asked about, the single flag still turns it on:
the edge forces `TraceBlocked` from `DetailedDownloadMetrics` in
`sdkResourceLimits` (§2b), and the worker should do the same in `peer.go` when
building its server config, so `--worker.storage.detailed-metrics` alone is
sufficient there too.

---

## 5. Consequences to communicate

- **The "Download Phases" Grafana dashboard goes blank in production** until
  someone sets the flags. 33 panels across
  `monitoring/grafana/dashboards/download/download-phases.json`. Add a note to
  `monitoring/README.md` and to `download-stats.md` §13 naming the exact flags.
- Enabling requires a restart (`docker compose restart edge`, or a worker
  rollout for the fleet).
- A worker with detail off still logs `download_id` at the interceptor level
  (`internal/grpcutil/log.go:97`) if a client sends the header - that is correct
  and self-gating, since clients only send it when their own flag is on.

---

## 6. Verification

**Unit**
```
cd /home/tuan/work/depin-workspace/go-sdk && go build ./... && go test ./... 
cd /home/tuan/work/depin-workspace/depin  && go build ./... && go test ./edgeserver/... ./worker/piecestore/... ./internal/p2putil/...
```
`go build ./...` in depin after every go-sdk edit is mandatory here - the local
`replace aioz-depin/go-sdk => ../go-sdk` (`go.mod:415`) means a go-sdk signature
change breaks depin silently in the editor but loudly at build.

Tests that must be updated, not just re-run:
- go-sdk `options_test.go:41-91` (per-option table), `:111-113` (zero-value),
  `client_internal_test.go:19-56` (field-copy regression).
- go-sdk `internal/segmentdownload/phases_test.go`,
  `internal/workerclient/steps_test.go`,
  `internal/piecedownload/admission_test.go`,
  `internal/metrics/observe_test.go`, `cmd/sdk-benchmark/phases_test.go` - each
  must construct the detail-on path explicitly.
- depin `edgeserver/metrics_test.go:156-184` - `recordSDKTransfer` becomes
  `sdkTransferSink(true)`.
- depin `worker/piecestore/endpoint_stats_test.go` - unchanged (it pokes atomics
  on a bare `&Endpoint{}`), but add a case asserting `Stats` is not chained when
  the flag is off.

**New tests (the actual guard)**

Add an off-path assertion in each repo - "flag off means the series never
appears" is the behaviour being shipped, and nothing currently tests it:
- `edgeserver`: a handler test that a `Config{}` (flag off) response carries **no**
  `X-Download-Id` header and fires no `edge_resolve_duration`.
- `worker/piecestore`: a `Download` test asserting `download_send_duration` is
  absent with `Config{DetailedMetrics: false}`.
- go-sdk: extend `monkit_download_test.go` to assert an empty snapshot and no
  `sdk_download_phase_*` when the client is built without the option.

**E2E** - `internal/testplanet/download_phases_test.go` is the load-bearing one.
It currently passes because tracing is unconditional; after this change it must
explicitly opt in on both sides:
- worker: `planet.config.Reconfigure.Worker` setting
  `cfg.Storage.DetailedMetrics = true` (testplanet binds with
  `cfgstruct.UseTestDefaults()`, `internal/testplanet/worker.go:70-78`).
- edge: `edgeserver.Config{DetailedDownloadMetrics: true}` at `:62-63` instead of
  the current `edgeserver.Config{}`.

Deliberately **not** using `testDefault:"true"` - leaving the production default
in force for every other test is what makes the off-path test above meaningful.

Then add the mirror-image e2e: same planet, flags off, assert none of the six
worker phase names and none of `edge_resolve_duration` /
`edge_transfer_duration` / `edge_download_ttfb` fired, and that the response has
no `X-Download-Id`.

```
cd /home/tuan/work/depin-workspace/depin
go test ./internal/testplanet/ -run 'TestDownloadPhaseTracing' -count=1 -v
```
Needs local Postgres: `postgres://depin:depin@127.0.0.1:15445/depin_test?sslmode=disable`
(container `depin-pgtest`). Note `TestRelayConcurrentPieceDownloads` in that
package is known flaky (~10%) and pre-existing.

**Manual smoke** - confirm the flag actually flips both ways:
```
./dev/bench/download-test.sh -a sdk -c 4 -n 8 -s 30MiB     # sdk-benchmark opts in: phases present
# then run an edge with the flag off and confirm curl -I shows no X-Download-Id,
# and /metrics on the edge has no edge_sdk_phase_* series.
```
