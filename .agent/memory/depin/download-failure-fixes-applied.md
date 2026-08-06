---
type: decision
tags: [depin, edge, go-sdk, download, pool, keepalive]
created: 2026-08-01
agent: main
---

Fixes applied for the edge/SDK download failures diagnosed in
[[edge-k6-download-failures]]. All TDD (failing test watched first).

**1. edgeserver: no more silent truncation.** `serveFull` streams chunked with
no Content-Length, so a mid-stream `DownloadByTicket` failure used to return a
clean 200 with a short body - undetectable by any HTTP client. It now
`panic(http.ErrAbortHandler)`s once bytes are already committed, so net/http
drops the connection without the terminating chunk and the caller's read
fails. Test `TestServeFull_ErrorAfterBytesWritten_ClientCanDetectTruncation`
(edgeserver/handler_test.go) drives a real `httptest.Server` + `http.Get`,
gating the failure on a channel so the client provably receives the leading
bytes first. The old test that asserted the truncated-200 was acceptable was
replaced. `serveRange` was deliberately left alone: it sets Content-Length, so
a short body is already detectable.

Test gotcha worth remembering: the writer chain embeds the
`http.ResponseWriter` **interface**, which does not declare `Flush`, so
`w.(http.Flusher)` silently fails. Use
`http.NewResponseController(rw).Flush()`, which follows the `Unwrap` chain.

**2. go-sdk pool: stop closing in-use connections.** The cache `Close`
callback's deferred-close guard was gated on
`LimitedReuseMaxStreams > 0 && inFlight > 0`, but `inFlight` was only ever
incremented in the limited-reuse branch - so reusable (direct) conns, which
`NewStream` hands back to the cache *while their stream is still running*,
were hard-closed on capacity eviction. Now: the reusable branch does
`acquire`/`release`, and the guard is `inFlight > 0 && !conn.Broken()` (broken
conns stay exempt - their streams are already dead, and deferring would leak
them; that exemption is what keeps `TestPool_Stale_EvictsBrokenConn` valid).
Also `NewDefaultConnectionPool` Capacity 200 -> 2000, matching what its own doc
comment already claimed. Tests: `TestPoolConn_Eviction_DoesNotClose
ReusableConnWithLiveStreams` and `..._ClosesReusableConnOnceStreamsDrain`.
Committed by the user as `7d35f63`.

**3. go-sdk: a download timeout now reports as a deadline.** New
`classifyDownloadErr(ctx, err)` in download.go, wired into `DownloadManifest`
and `DownloadManifestRange` via a deferred named-return (registered *after*
`defer cancel()` so it runs *before* it). When the bound from
`withDownloadTimeout` expires, the teardown races the work, so the surfacing
error was an incidental symptom (`eestream: read completed buffer`) and
edgeserver mapped it to 502 instead of 504. Fixed at the boundary that owns the
timeout rather than inside eestream - one place, covers every inner failure
mode. Consistent with [[sdk-owns-safety-defaults]].

**4. coord keepalive: NO CODE CHANGE NEEDED.** `coord/server/server.go:459`
already sets `MinTime: 5s, PermitWithoutStream: true`. The observed
`GoAway ENHANCE_YOUR_CALM "too_many_pings"` comes from the **deployed fleet
workers**: they run `v0.0.1-116-gb82a3b6-dirty`, built from a 2026-07-24
commit, and `git merge-base --is-ancestor` confirms that predates the worker
keepalive fix `267439a` (2026-07-30). Remedy is a fleet redeploy, not code.
(The unused `internal/grpcutil/server_builder.go` and `cmd/keytool`'s sign
server also lack the policy, but neither is on the download path.)

**Verification outcome - honest state.** Single download healthy (hash ok,
3.9 MiB/s). The self-inflicted failure is gone: `grpc: the client connection is
closing` went 5931 -> **0** in the same 20-way reproduction. But 20-way
concurrency still fails end to end, now dominated by a *different*, previously
masked error: **650x `keepalive ping failed to receive ACK within timeout`**.
That is the go-sdk client keepalive in `internal/grpcutil/grpcconn/conn.go`
(`Time: 10s, Timeout: 3s`) - a 3-second ping-ACK budget is far too tight on a
congested relay circuit, where a PING queues behind megabytes of DATA frames.
**FIXED 2026-08-01**: raised to 20s (grpc-go's own default, and what the worker
server already uses). Full-ladder A/B on the live fleet, 30 MiB files,
hash-verified - before: 27/61 OK, 751 keepalive failures, 1172 stream resets;
after: **61/61 OK, zero errors of any kind at every level 1..20**. The 12/16/20
levels went 2/12 -> 12/12, 0/16 -> 16/16, 12/20 -> 20/20. A ping-ACK budget is
not a liveness measure on a busy connection: the PING queues behind in-flight
DATA frames, so on a saturated relay it was killing healthy connections and
taking every multiplexed piece stream with them.

Note the 20x256 MiB case is simply over capacity (5 GB over one relay at
~6 MB/s is ~14 min vs the 2-minute bound), so `DeadlineExceeded` there is
correct behaviour, not a bug.


## edgeserver log shipping (2026-08-02)

edgeserver had `Metrics telemetry.Config` (metrics -> Prometheus) but **no
logship at all** - `grep -rl logship --include=*.go` hit only coord, coord/relay,
worker and pkg/logship. That is why the edge's `"download failed"` line was
unreadable during the 256 MiB investigation: its logs only ever went to stdout.

Wired it mirroring `coord/peer.go:112-131`:
- `edgeserver.Config.LogShip logship.Config`
- `newLogShipper(logger, cfg)` -> `(*zap.Logger, *logship.Client)`, teeing
  `logship.NewCore` into the zap core and handing the client the PRE-wrap logger
  so a push failure cannot feed back into the buffer that just failed to flush.
  A `NewClient` error only warns - log shipping is observability, not function.
- Called **first** in `New()`, before the SDK client / handler / debug server
  capture the logger by value, otherwise the wrap never reaches them.
- `Run` runs it; shutdown closes it **after** telemetry so its final flush
  catches shutdown-time logs.

Tests (RED first): `TestNewLogShipper_ShipsLogsToLoki` drives a real
`httptest.Server` as Loki, uses `FlushInterval: time.Hour` so the only push is
`Close()`'s final flush (deterministic, not timing-dependent), and asserts the
message + field reach the endpoint; `TestNewLogShipper_DisabledLeavesLoggerAlone`
asserts an empty URL returns the same logger and a nil client.

Deploy: `log-ship.url` and `log-ship.bearer-token` are `noflag:"true"`, so they
are NOT written by `setup` and are not CLI flags - they must come from
config.yaml/secrets.yaml or `AIOZ_LOG_SHIP_URL` / `AIOZ_LOG_SHIP_BEARER_TOKEN`
(viper AutomaticEnv, prefix AIOZ). Also set `log-ship.app=edge` and
`log-ship.role=edge-server1` - App defaults to "coord", which would file the
edge's logs under the coordinator's streams.
