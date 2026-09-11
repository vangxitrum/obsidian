---
type: decision
tags: [depin, download, metrics, monkit, observability, edge, worker, relay, go-sdk, grafana]
created: 2026-08-18
agent: main
---

# Download phase tracing across edge, go-sdk, worker, relay

Built to answer "which part of a download is slow" - previously unanswerable
because `piece_open` was one opaque number and there was no way to follow one
request across components. Branch `feat/improve-download-speed`, uncommitted in
BOTH repos (`depin` and `../go-sdk`).

## What the instrumentation now says

- `piece_open` split into `piece_sign_order` / `piece_dial` /
  `piece_stream_open` / `piece_request_send`, plus a NEW `piece_first_byte`
  (the wait for the worker's first frame, which previously belonged to no phase
  at all because `piece_open` ended when the request frame was sent).
- `admission_wait` (tagged `stage=global|window`) - queueing for piece-stream
  permission before any worker is contacted. Prior investigations blamed ~12.4s
  of a 13.45s latency increase on pre-piece queueing with no metric to prove it.
- `client_write` - time blocked handing plaintext to the consumer.
- `sdk_coord_resolve_duration` - the coordinator resolve, which was measured
  NOWHERE (the collector is created after it, and so is the sink's wall clock).
- `sdk_*` monkit family emitted by the SDK itself, so a direct SDK download
  (cmd/sdk-benchmark) is measured without registering a MetricsSink.
- Worker serve path: `download_{verify_orderlimit,verify_order,begin_save_order,
  reader_open,order_wait,disk_read,send}_duration`. `order_wait` is the worker
  BLOCKED ON THE CLIENT's progressive order authorization - from outside it is
  indistinguishable from a slow disk.
- `p2p_resource_blocked{kind,scope}` - libp2p resource-manager refusals, in
  BOTH repos' `internal/p2putil`. Previously silent; from the far end a blocked
  stream looks like an open that failed or hung for no reason.
- `X-Download-Id` correlation id: edge mints it, SDK attaches it as gRPC
  metadata (`x-aioz-download-id`) to the coord resolve and every piece stream,
  worker interceptors adopt it into the zap log context.

## Load-bearing findings

- **`transport` must be classified AFTER the dial, not from the address.** The
  coordinator can hand out several addresses and libp2p's dial ranker chooses
  among them, so the address does not say whether a piece went over the relay.
  Classification comes from `p2pmonitor.Observer.Transport(peerID)`, which reads
  the live per-peer conn state (`network.Stats.Limited`).
- **Two planned relay metrics are NOT implementable.** libp2p's
  `circuitrelay.MetricsTracer` gives `ConnectionRequestHandled(status)`,
  `ConnectionOpened()`, `ConnectionClosed(d)` and `BytesTransferred(n)` with NO
  per-circuit identity, so `relay_circuit_open_duration` (no request→open
  correlation) and per-circuit `relay_circuit_bytes` cannot be derived.
  `p2p_resource_blocked` was built instead, and it covers relay, worker and edge.
- **The ranged download path was lying about CPU.** It reader-timed both
  `erasure_decode` and `decrypt`, so network waiting was reported as cipher work
  - the exact bug the whole-file path was fixed for, left in place on the ranged
  one. One phase name meant two different things depending on whether the caller
  asked for a range. Fixed by moving it onto `Collector.TransformReader`.
  Regression test `TestRangedPathAttributesLikeWholeFilePath` was causally
  verified (fails against the old code).
- **`testplanet`'s edge registers no MetricsSink.** `edgeserver.NewFromClientConfig`
  shares the uplink's already-built SDK client; only production
  `edgeserver.New` calls `WithMetricsSink`. So `edge_sdk_phase_*` CANNOT be
  asserted in testplanet - assert `sdk_download_phase_*` there and cover the
  edge mapping in `TestRecordSDKTransfer_NewPhases`. Same class of trap as
  [[testplanet-coverage-plan]]'s false-green metadata test.
- **`e.liveRequest` counts uploads too**, so a download-only gauge could not be
  derived from it; added a separate `liveDownload` and a `piecestore_live_requests`
  StatSource with `total`/`download` fields.
- **`HashStoreBackend.Stats` was never registered.** It built a proper monkit
  series that reached neither `/metrics` nor the push client - no `mon.Chain`
  anywhere. Now chained in `worker/peer.go`.
- Pre-existing, unrelated, fixed to unblock the suite: go-sdk
  `internal/grpcutil/connector/p2p_connector_test.go` did not compile - a
  committed signature change added a `disableHolePunching bool` and the test was
  never updated.

## Gotchas for next time

- `Collector.Observe` now owns which phases sum and which keep the max
  (`piece_slowest`). Producers must not call Add/Max directly, or the two
  download paths drift apart again.
- `piece_download` is a UNION of concurrent reads, not a sum - never stack it
  against per-piece phases without saying so.
- Queue-depth accounting: increment BEFORE handing a buffer to the consumer
  goroutine, or the consumer decrements first and the peak is understated.
- `testplanet.RunWithLogger` was added so a test can observe peer logs; the
  correlation assertion is the only thing holding the two repos' header-name
  strings together (causally verified by breaking the constant).
- Local test Postgres: `postgres://depin:depin@127.0.0.1:15445/depin_test?sslmode=disable`
  (container `depin-pgtest`).

Related: [[download-network-cap-root-cause]], [[gosdk-streaming-reconstruction]],
[[edge-request-admission]], [[edge-load-test-production-contention]],
[[worker-holepunch-never-upgrades]], [[download-benchmark-harness]],
[[coord-resolve-performance]].

## Corrections and later fixes (same session, 2026-08-18)

- **`piece_dial` measured nothing.** `pool.Pool.Get` returns a LAZY `poolConn`;
  the real connect (libp2p, relay circuit, TLS, gRPC) happens on the first
  `NewStream`, i.e. inside what was labelled `piece_stream_open`. Reading
  `piece_dial` as "the dial is free" is the exact wrong conclusion. Fixed with
  `pool.WithStreamTimer` (passed by CONTEXT, because the call goes through
  generated gRPC code with no seam for a callback), splitting
  `piece_conn_acquire` from `piece_stream_open`. Guarded by a test where a 40ms
  fake dial must land in conn_acquire and must NOT appear in stream_open.
- **`transport` was classified too early** - right after the lazy DialNodeURL,
  so a cold piece had no libp2p conn to inspect and got tagged `unknown` purely
  for being cold. The tag read warm-vs-cold, not relay-vs-direct. Now classified
  at each emission site, after the connection exists.
- **Three dashboard panels were broken**: `sum by (__name__) (rate({__name__=~...}))`
  fails outright - rate() STRIPS `__name__`, so matched series collapse to
  identical labelsets ("vector cannot contain metrics with the same labelset").
  Fixed with explicit per-phase targets. Any multi-metric regex inside rate()
  has this problem.
- **Phase names now have one home**: `uplinksdk.DownloadPhases()` + exported
  `Phase*` constants, cross-checked against the producers' own constants in
  `monkit_download_test.go`. The edge test and the e2e range over it instead of
  repeating a literal list (a copied list keeps passing after a rename).
- **`erasure_decode` has no emitter** - removed from the dashboard stack; RS
  decode is pull-driven so timing it reports network waiting as CPU, and the
  wait is recorded once as `pipeline_stall`.
- **Duplicate `mon.Chain`**: guarded with sync.Once. A chained source emits a
  fixed series key with no per-peer tag, so N workers in one process (testplanet)
  emitted N identical series - duplicate samples at the same timestamp, which
  Prometheus rejects.
- **monitoring/.env had drifted from .env.example**: `WORKER_METRICS_BEARER_TOKEN`
  (hard-required, `:?`) plus LOKI_RETENTION/PROM_RETENTION_SIZE were missing, so
  EVERY `docker compose` command against monitoring/ failed before doing
  anything. Worse, Caddy's `@write_worker` matcher expands an unset var to the
  literal `Bearer ` - an empty token is an ACCEPT rule. Local stack only
  (metrics.localhost, 0 requests through Caddy), but PRODUCTION must be checked.
  Note: `docker compose restart grafana` cannot be used to reload provisioning
  while any required var is missing; `docker restart monitoring-grafana-1` works.
- Grafana reads provisioning ONLY at startup, so a new dashboard subdirectory
  needs a container restart; dashboard JSON itself reloads every 30s.

## The flaky testplanet tests were NOT caused by this work

Measured, not assumed. `TestDownloadSurvivesWorkersDyingMidStream`:
HEAD go-sdk = 17/25 pass (32% fail); with these changes = 21/25. An earlier
"10/10 baseline" was misleading because depin's committed go.mod pins the
PUBLISHED go-sdk, while the working tree replaces it with `../go-sdk` - so a
naive baseline tests a different SDK than the one you are changing. Always
point the baseline worktree at `../go-sdk` HEAD.

Root cause of the flake, now fixed (both tests, 15/25 -> 25/25 and 15/15):
they sized the worker kill from ONE sampled segment (`AuditSegmentForFile` is
an unordered `LIMIT 1`). Placement is per segment and a segment can commit with
fewer pieces than the scheme's total, so another segment could be left below
RequiredShares - the coordinator then refuses the re-resolve with
"only 1 of 2 required pieces are available", for a segment the test never
looked at. Fixed with `Coord.AuditSegmentsForFile` (all segments) plus an
`expendableWorkers` helper that only kills workers every segment can spare; the
unrecoverable-case sibling now leaves `RequiredShares-1` workers alive planet
wide. Still observed once each and not reproduced in 2 later full runs:
`TestOrderAmountReflectsBytesServed`, `TestRelayConcurrentPieceDownloads`.

## The edge read-ahead buffer was inert (found + fixed 2026-08-19)

`edgeserver/readahead.go` shipped with a queue sized in CHUNKS
(`make(chan []byte, budget/readAheadChunk)`) but fed one Write at a time, so a
caller writing less than a chunk pinned a whole slot per write and the buffer
collapsed to `writeSize * slots`.

The go-sdk writes **240 bytes** per call - `transformedReader.Read`
(pkg/encryption/transform.go:90) fills its buffer with exactly ONE AES-GCM block
and returns it regardless of how big `p` is, and DefaultBlockSize 256 minus the
16-byte GCM tag = 240. Confirmed by arithmetic on live data: 30 MiB / 131072
(the observed `decrypt` count) = 240.0 bytes exactly.

Consequence: a nominally 8 MiB buffer really held **4.8 KiB** - measured 1748x
short - so the SDK stayed coupled to the client socket, which is the exact
coupling the type was written to remove. That is why `edge_readahead_blocked`
was 7.2 s/download with `edge_readahead_peak_bytes` at ~0: the producer was
blocking on a queue that filled after ~20 writes, NOT paying per-write overhead
(an early wrong guess - at 240 B the writer still does 264 MB/s against an idle
consumer, so overhead alone cannot explain 7.2 s).

Fix (option B of two; A was to make transformedReader fill `p` with multiple
blocks): Write now accumulates into a chunk and hands it over when the chunk is
full OR when `len(ch) == 0`. The idle condition is what stops it behaving like
bufio - holding data back only while the consumer already has work queued means
it never starves, so first-byte latency for HLS is preserved. close() flushes
the partial chunk (the tail of every transfer).

Causally verified: the guard test reaches 7,920 B peak against the old code
(= 33 slots x 240 B) and >= 4 MiB with the fix.

Measured writer throughput by write size (idle consumer): 240 B 264 MB/s,
4 KiB 648, 32 KiB 2369, 256 KiB 5395 - 20.4x.

NOT yet measured in production: the edge must be redeployed to see the effect.
Live before-fix numbers for comparison (instance 8d6257ba30c6, new-build edge,
21 downloads): edge arm 2.61 MiB/s vs direct-SDK arm 10.91 MiB/s;
client_write 9.58 s and readahead_blocked 7.22 s per 23 s download.

## Round-trip investigation (Phase 1 instrumentation, 2026-08-19)

After the buffer fix, the remaining cost is the worker blocked in `Send`:
**322.6 ms for a 12 KiB piece**, against 0.73 ms of transmission at 16 MiB/s
(442x). Worker answers in 1.08 ms; disk 0.09 ms. Request total ~= piece_slowest.

**Ruled out with numbers, so nobody re-litigates:**
- *The relay.* Edge, coord and relay are ALL co-located on 68.183.189.51, so the
  edge->relay leg is local and the 40.8 ms RTT is the relay->worker internet leg
  - which a direct link would traverse identically. 8 x 40.83 ms = 326.6 ms
  predicted vs 322.6 ms measured, closing to 1.2%. Relay carries 0.23 MiB/s
  across 190 circuits. Removing it buys ~4 ms of 322 ms. Reverse-dial was
  DESIGNED AND REJECTED on this evidence.
- *Long-tail abandonment.* 120,558 worker successes, ZERO cancels, 221 failures.
- *Connection setup.* 91% pool hit rate.
- *The client.* Read-ahead blocks 21 ms/download after the coalescing fix.

**Reverse-dial feasibility (researched, not built).** libp2p WOULD cooperate:
`swarm.bestConnToPeer` is direction-agnostic, and gRPC/TLS roles are bound to
who opens the STREAM, not who dialed the connection (`tls.Client` per stream at
p2p_connector.go:189 vs the worker's `tls.NewListener`). Two structural blockers:
the edge installs no `SetStreamHandler` (`NewLibP2PListener` has ZERO callers in
go-sdk) and the worker's dialer is TCP-only (`worker/peer.go:182-185` uses
`NewDefaultTCPConnector`, so its libp2p host only serves inbound + reserves a
relay circuit). Also `Reserved()`/`Replay()` in connector/dial.go are dead code -
`lowLevelDial` never passes DialOptions.

**Edge p2p port 7790 is CLOSED.** Announced as
`/ip4/68.183.189.51/tcp/7790` but unreachable from both fleet hosts and locally,
while 7777 (coord) and 7781 (relay) on the same host answer. It appears in no
compose file or Dockerfile - only hand-edited config. This alone explains 99.8%
relayed and why DCUtR upgrades are rare.

**Phase 1 instrumentation added (this session):**
- `download_send_duration` now tagged `size_bucket`
  (0-16KiB/16-128KiB/128KiB-1MiB/1MiB+). THE discriminator: flat across buckets
  = round trips; rising = window/bandwidth.
- `download_send_calls` + `download_send_max_duration` separate "one long block"
  from "many short ones" (sendWait was a sum over the chunk loop).
- `sdk_piece_recv_gap_duration` = interval between one Recv returning and the
  next being issued, i.e. CONSUMER time, to test whether the worker blocks
  because the edge is slow to drain.

**Gotcha found by the e2e:** `piece_recv_gap` only exists for a piece spanning
MORE THAN ONE frame, and a 12 KiB piece fits one 64 KiB chunk - so it never
fires in production or testplanet. It is therefore excluded from
`DownloadPhases()` (whose contract is "these always fire") and charted on its
own panel, not stacked. Its absence is itself the finding: a one-frame piece has
no consumer-side gap, so the reader cannot be why the worker blocks on small
pieces.

**Known flaky, PRE-EXISTING:** `TestRelayConcurrentPieceDownloads` fails ~10% in
isolation (9/10), unrelated to this work.

## Phase 1 RESULT: it is the relay's per-circuit buffer (2026-08-19)

Rolled worker **v0.0.10** with `fleet.sh rollout v0.0.10 100 144` - a ~10 min
ramp. Use a non-zero safe_rate ALWAYS: cursor=100 restarts the whole fleet at
once and cost a 0/8 benchmark on v0.0.9. Ramped adoption went 4 -> 100 of 150
workers over 21 min with the fleet serving 33 pieces/s throughout.

**The size_bucket split answered it:**

| bucket | mean send | pieces |
|---|---|---|
| 0-16KiB | **0.0 ms** | 30,374 |
| 1MiB+ | **4,529.8 ms** | 188 |

NOT flat -> the cost is window/bandwidth, NOT round trips. My round-trip framing
was wrong, and so was the "442x overhead on a 12 KiB piece" line: **322.6 ms was
a BLENDED mean**. Production 12 KiB pieces cost ~0 ms in Send; all the time was
in the rare 1 MiB pieces. Never quote a blended mean over a bimodal population.

**Cause: `Resources.BufferSize` on the relay (libp2p default 2048, depin 8192).**
go-libp2p `p2p/protocol/circuitv2/relay/relay.go:510` and `:536` do
`buf := pool.Get(r.rc.BufferSize)` then `copyWithBuffer(dest, src, buf)` - the
relay reads at most ONE buffer from the source before writing it onward, so a
relayed stream is store-and-forward capped at **buffer / RTT**:

- 8 KiB / 40.8 ms = **196 KiB/s** predicted; **249 KiB/s** measured (within 27%)
- `download_send_calls` 1.104 = ~1 Send for small pieces, ~16 for 1 MiB
- 257 ms per 64 KiB chunk
- yamux window would allow 6.1 MiB/s - 27x above observed, so NOT binding
- relay also reserves `2 x BufferSize` per circuit (relay.go:276)

**Fix written, NOT deployed** (relay host unreachable from this session):
`configs/relay/config.yaml` -> `resources.buffer-size: 262144`. Lifts the
ceiling 32x to 6.13 MiB/s, which is exactly where the endpoints' yamux window
sits, so it stops being binding without overshooting the next constraint. ~140
MiB RSS at the 279 peak circuits; relay runs `resource-limits.unlimited: true`.
To go higher, raise `resource-limits.stream-window` on WORKERS and EDGE (relay
is already 16 MiB).

Dashboard panel "Per-stream throughput vs the relay's store-and-forward ceiling"
plots measured KiB/s against both buffer ceilings - the before/after instrument.

**Measurement 3 (`sdk_piece_recv_gap_duration`) still has no data**: only the
workers were rolled out; it lives in the go-sdk and needs an EDGE redeploy. So
"the edge is slow to drain the stream" is not yet excluded as a contributor.

## Gated behind config (2026-08-20, same branch, still uncommitted)

All of the above is now **off by default** and turned on per component:

| Component | Flag | Env |
|---|---|---|
| Worker | `--worker.storage.detailed-metrics` | `AIOZ_WORKER_STORAGE_DETAILED_METRICS` |
| Edge (also drives its SDK client) | `--detailed-download-metrics` | `AIOZ_DETAILED_DOWNLOAD_METRICS` |
| Direct SDK embedder | `uplinksdk.WithDetailedMetrics(bool)` | - |
| Any libp2p role, refusals only | `--resource-limits.trace-blocked` | `AIOZ_..._RESOURCE_LIMITS_TRACE_BLOCKED` |

Startup-only (cfgstruct), so enabling needs a restart or a fleet rollout. The
edge and worker flags imply `trace-blocked` for their own host; the relay needs
it separately because it has no download path to hang a flag on.
`cmd/sdk-benchmark` enables detail unconditionally - producing the breakdown is
what it is for.

Always-on regardless: `edge_download_*`, `edge_sdk_fetch_*`,
`sdk_download_duration`/`_bytes`/`_speed_bps`, retry counters, the worker's
started/success/failure/cancel counters and byte totals, and
`download_time_to_first_byte_sent`. The read-ahead BUFFER also stays on - it is
a throughput fix, only its accounting is instrumentation.

### The two traps this work turned on

- **A method value taken from a nil pointer is NOT nil.** `coll.Observe` on a
  nil `*metrics.Collector` and `m.observe` on a Manager with no timer both
  satisfy every `!= nil` gate downstream, so "tracing is off" silently stayed
  open and the untraced path kept minting `sdk_worker_piece_duration` (one
  series per worker in the fleet). Fixed with `Collector.ObserveFunc()` and
  `Manager.observeFunc()`, which return a genuine nil. Any seam meaning "is a
  timer configured" must go through those, never a bare method value. The same
  shape bit `workerclient.Config.StepTimer`.
- **monkit keys a series by name AND tags.** `scope.DurationVal("edge_resolve_duration")`
  with no tags addresses a DIFFERENT, always-empty series than the tagged one
  the code writes to - so asserting zero against it passes whether or not the
  metric fired. Caught it because the paired "on" assertion also read zero.
  `durValCountTagged` / `resolveCount` in `edgeserver/metrics_test.go` exist for
  this; the same hazard applies to any monkit assertion in this repo.

### Structure

Gate on the seam that already exists rather than adding a parallel flag: nil
`*metrics.Collector` silences `client_write` / `piece_download` / `decrypt` /
`pipeline_stall` / `piece_open` / `piece_slowest` / `admission_wait` with no
branch at any of those call sites. `workerclient.Config.Detailed` was added
only because its monkit half had no seam. Edge uses a `detail bool` type
threaded from `Server.cfg` - NOT a package global, because testplanet builds
several edge servers per process from literal `Config` values. Worker uses
`phaseTimer` on `Endpoint.config` (never `Service.config`, which is never set).

`testplanet.Config.DetailedDownloadMetrics` turns it on planet-wide (workers +
uplink SDK clients); the edge is separate because `NewFromClientConfig` shares a
caller-built client and cannot pass SDK options.

`TestDownloadPhaseTracingOffByDefault` is the guard, and was causally verified
against all three gates independently (force worker `phaseTimer(true)`, edge
`detail(true)`, SDK `cfg.Detailed = true` - each produced a distinct failure).

### go-sdk published, depin repinned (2026-08-20)

go-sdk `f628fd9` (`feat(download): add detailed phase breakdown for latency
attribution`) carries the tracing AND its gate, and is on `origin/main`. depin's
`go.mod` dropped the `replace aioz-depin/go-sdk => ../go-sdk` and went back to
the published form:

```
replace aioz-depin/go-sdk => gitlab.internal/aioz-depin/go-sdk v0.0.0-20260820072243-f628fd9877d8
```

**Pseudo-version timestamps are UTC, not local.** The fleet commits at +0700, so
`git show -s --date=format-local` gives the wrong stamp; use `TZ=UTC0`. Verified
against the two prior pins (37f1de6 -> 20260814082023, 13b7d8d -> 20260805041745)
before writing the new one.

`go mod download <path>` on the REPLACE TARGET reports "not a known dependency"
- the target is not a require. Just `go build ./...`, then
`go mod download aioz-depin/go-sdk` (the require path) to fill go.sum.

Proof that matters, because CI has no `../go-sdk`: `mv ../go-sdk` away, then
`go build ./...` and `go test ./edgeserver/...` from the depin dir. Run it from
the module dir - running `go build ./...` from `depin-workspace/` instead exits
0 with "directory prefix . does not contain main module", which reads like a
pass.

The `// indirect` churn in go.mod (go-grpc-middleware, go-yamux/v5,
golang.org/x/term, blake3) is NOT from the repin - `go mod tidy` reproduces it
exactly. HEAD's go.mod was simply stale after 0bf6aa3 added the yamux import.
