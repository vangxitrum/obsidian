---
type: fact
tags: [depin, edge, benchmark, k6, relay, download]
created: 2026-07-31
agent: main
---

Diagnosis of "download failed" in the `dev/bench/pipeline.sh` edge k6 runs
(testids `edge-20260731-154837` and `edge-20260731-161332`).

**What the failures actually are.** Queried Prometheus (`k6_http_reqs_total`
carries a `status` label):
- run 1: 67x 502, 34x 504, 7x 200
- run 2: 92x 502, 31x 504, 0x 200

Both statuses are emitted by the edge itself, not by any proxy -
`edgeserver/handler.go` `writeDownloadError`: `DeadlineExceeded` -> 504
"download timed out", anything else -> 502 "download failed". Confirmed by
`edge_download_error` incrementing in the same window. There is no reverse
proxy in front of `edgeserver1.appdemo.cyou` (root path returns a bare Go
404, no `Server:` header).

**ROOT CAUSE (proven 2026-07-31, instrumented reproduction): the go-sdk
connection pool hard-closes healthy, in-use connections on capacity eviction.**

`internal/grpcutil/pool/conn.go` `NewStream` returns a *reusable* (direct,
multiplexable) conn to the cache **immediately after opening a stream**, while
that stream is still running. `pool.go`'s cache `Close` callback has a guard
that defers closing a conn with live streams - but it is gated on
`opts.LimitedReuseMaxStreams > 0 && pv.inFlight.Load() > 0`, and `inFlight` is
only ever incremented in the limited-reuse branch (`p.acquire`). So for
reusable conns `inFlight` is permanently 0, the guard never fires, and
`cache.Put`'s capacity eviction calls `pv.conn.Close()` on a connection with
N live streams. Every stream on it then fails with
`grpc: the client connection is closing`.

Compounding: `internal/dial/dial.go` `NewDefaultConnectionPool` sets
`Capacity: 200` while its own doc comment above it says the values were raised
to "2000/32". 20 concurrent downloads x 29 pieces x prefetch depth 2 needs
~1160 simultaneous conns, so the cache is permanently over capacity and evicts
continuously.

Instrumented proof (temporary stack dump in the Close callback, since
reverted): 2420 closes in one run, **100% `reusable=true broken=false`**, and
every stack was `cache.Put -> closeEntry -> Options.close -> pool.New.func`
(the capacity-eviction path) - never expiration, Take-stale, or `pool.Close`.
Result: 20/20 downloads failed, always at `open segment 1`, sole error
`grpc: the client connection is closing` (1107x).

CORRECTION: an earlier reading of these logs treated "open segment 1" as the
*second* segment and explained it via prefetch depth 2. That was wrong. The
number in `sdk: open segment %d` is the coordinator's `seg.SegmentNumber`,
which is **1-based** - a single-segment 30 MiB file also fails at "segment 1".
So the failures are at the FIRST segment, and prefetch depth explains nothing
about them.

Why the signature fits the fleet observations: fails only under concurrency
(needs >200 conns in flight); mass simultaneous stream
deaths of *varying ages* but with a hard **minimum-age floor** (~3.2 s on the
fleet) - LRU evicts oldest first, so the youngest conns always survive.
Relay circuit closures (95) were an order of magnitude below transport deaths
(686) in the same window, so the relay circuit was never the problem.

CAVEAT not yet closed: the local reproduction reaches workers over **direct**
conns (client and the 50 fleet containers share a host), so `Limited()` is
false and they take the reusable path. The deployed edge reaches workers over
**relayed** conns, which with reuse off are checked out for the stream's
lifetime and so are not in the cache to be evicted. Same signature, but confirm
the edge's `server.relay-reuse-max-streams` before assuming one bug explains
both.

**Symptom chain under edge concurrency.** Reproduced
with `sdk-benchmark` against coord while a k6 run was live:

```
sdk: open segment 1: piecedownload: fetched 12 pieces, need 29:
  rpc error: code = Unavailable desc = ... stream reset by remote, error code: 0;
  rpc error: code = Unavailable desc = keepalive ping failed to receive ACK within timeout;
  rpc error: code = DeadlineExceeded ...
```

Segment 0 completes, segment 1 cannot reach the RS threshold. So:
- failure before the first byte -> 502
- failure after 2m (`go-sdk` `DefaultDownloadTimeout`, edge
  `ServerConfig.DownloadTimeout` defaults to 0 = use SDK default) -> 504
- failure after bytes already streamed -> **HTTP 200 with a truncated body**
  (`serveFull` cannot change the status once `cw.written > 0`, it only logs).
  Observed a 256 MiB file returning exactly 67108864 B (= one
  `coord/file` `DefaultSegmentSize`) with status 200. Silent corruption; a
  plain GET sets no Content-Length, so the response is chunked and looks clean.

Not structural: idle, the same 256 MiB file downloads whole, hash ok, 41s,
6.3 MiB/s. Purely load-induced. See [[grpc-pool-reuse-relay-regression]] and
[[relay-conn-reuse-ws2]] - measured reuse ratio was 0.36 / 0.53 during the runs,
so limited-conn reuse was active.

Throughput collapse: 6.3 MiB/s at 1 stream -> ~0.5 MB/s per download at 32 VUs
(`edge_download_speed_bps` ravg 508880), `edge_sdk_phase_piece_download` 38.5 s.

**Re-test against the live edge with the system otherwise idle (2026-07-31
~17:00), no k6 and no pipeline running:**

| case | result |
|---|---|
| 30 MiB (1 segment), 1 concurrent | 200, full 31457280 B, ~5-7 s |
| 30 MiB, 8 concurrent | 5x 200, **3x 502** |
| 30 MiB, 20 concurrent | **20/20 502** |
| 256 MiB (4 segments), 1 concurrent | 200 but only **67108864 B = 1 segment**, in 28.7 s |
| 256 MiB, Range at offset 200,000,000 | 206, `content-range: .../268435456` - data intact |

Two separate defects, both reproducible with zero load:
1. **Concurrency**: the edge 502s from ~8 concurrent downloads even on
   single-segment files, so this is not the multi-segment path.
2. **Silent truncation of whole-file multi-segment downloads**: `serveFull`
   sends `transfer-encoding: chunked` with **no Content-Length**, so when
   `DownloadByTicket` dies mid-stream the client receives a clean-looking
   200 with a short body. No HTTP client can detect it.

Calling the SDK's `DownloadByTicket` directly (throwaway `cmd/tickettest`,
since deleted) on the *same ticket*, bypassing the edge entirely:
`bytes=180376448 elapsed=2m0.95s err=sdk: stream segment 3: eestream:
unexpected error: eestream: read completed buffer` - i.e. it ran into the
2-minute `DefaultDownloadTimeout` (180 MB / 121 s = 1.5 MB/s; 256 MiB needs
~180 s) and surfaced a confusing eestream error instead of DeadlineExceeded,
which is why edgeserver maps it to 502 "download failed" rather than 504.
The run also logged repeated `GoAway ... ENHANCE_YOUR_CALM "too_many_pings"`
against **coord** - the SDK pings every 10 s and coord's enforcement policy
still rejects that, unlike the worker side which was fixed. See
[[project_coord_keepalive_bigupload]].

**Harness bugs found (both worth fixing before trusting another run):**
1. `dev/bench/pipeline.sh` `cmd_run` passes `VUS`/`DURATION` as *environment*
   variables to `run-download-case.sh`, but that wrapper only forwards
   *positional* `KEY=VALUE` args into the k6 pod env. So `VUS=64 DURATION=3m`
   is silently dropped and k6 runs the script defaults (32 VUs, 1m) -
   `vus_max` in both reports is 32, not 64. Fix: pass them as args
   (`./tests/run-download-case.sh VUS=$VUS DURATION=$DURATION`).
2. `dev/bench/report.sh` labels k6 trend values "ms". The k6 Prometheus
   remote-write output pushes time trends in **seconds** - `duration p95
   50.15` is 50 s, not 50 ms. Cross-check: 32 VUs x 178 s / 108 iterations
   ~= 53 s per request, and `k6_iteration_duration_avg` = 36.9.
3. `report.sh` reports `failed rate` as `max_over_time(...)` of a per-interval
   rate, so it prints 1.00 whenever any single 5s window was all-failures.
   The true run-1 ratio was 101/108 = 0.935. Compute it from
   `k6_http_reqs_total` split by `status` instead.
4. `test-pipeline.sh` sets `REMOTE_HOST='demo'`, which is 68.183.189.51 - the
   **same host as edgeserver1**. The k6 load generator competes for CPU with
   the edge's RS decode (`edge_sdk_phase_erasure_decode` ~9 s). Move the
   loadgen off the edge host before drawing throughput conclusions.

Edge under test: `v0.0.1-124-g9509b45-dirty`; workers `50x
v0.0.1-116-gb82a3b6-dirty`. Edge logs are NOT shipped to Loki (only `app=coord`
roles api/core/ranged-loop/relay are), so status attribution had to come from
Prometheus + local reproduction.
