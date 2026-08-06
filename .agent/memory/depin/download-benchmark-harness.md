---
type: reference
tags: [depin, benchmark, grafana, prometheus, download]
created: 2026-08-01
agent: main
---

Self-service download benchmark: `dev/bench/download-test.sh` +
Grafana **Testing -> Download Benchmark** (`uid: download-benchmark`).

```bash
./dev/bench/download-test.sh                          # sdk arm, 8 concurrent, 30MiB
./dev/bench/download-test.sh -a both -c 16 -n 16      # sdk + edge under one testid
./dev/bench/download-test.sh -L "1 4 8 12 16 20"      # concurrency ladder
./dev/bench/download-test.sh --seed 5 -s 256MiB       # upload a corpus first
./dev/bench/download-test.sh --no-push                # numbers only, publish nothing
```

Two arms: `sdk` (go-sdk client straight to workers via coord+relay - the path the
SDK fixes affect) and `edge` (plain concurrent HTTP GETs of presigned links).
The edge arm counts a **200 with a short body as a failure**, which is what
catches the truncation bug from [[edge-k6-download-failures]].

Deliberately does NOT use the k6 pipeline: that needs a remote k3d cluster and
carries the harness bugs noted in [[edge-k6-download-failures]] (VUS/DURATION
silently dropped, loadgen co-located with the edge host).

**Transport: Pushgateway.** A benchmark run is a short-lived batch job, so there
is nothing to scrape when it ends. Added `pushgateway` (prom/pushgateway:v1.9.0,
bound `127.0.0.1:9091`) to `monitoring/docker-compose.yml` plus a `pushgateway`
scrape job in `prometheus/prometheus.yml`. **`honor_labels: true` is required** -
without it the scrape overwrites the run's own `job`/`testid` grouping labels.
Metrics published: `dl_bench_downloads{,_ok,_failed}`,
`dl_bench_throughput_mibps`, `dl_bench_duration_seconds`, and
`dl_bench_errors{kind=keepalive_ack_timeout|stream_reset|conn_closing|deadline_exceeded}`,
all labelled `testid`, `arm`, `size`, `conc`.

**Grafana gotcha that cost real time:** a `barchart` panel needs the Prometheus
labels lifted into columns (`format:"table"` + `joinByField`), and that
conversion happens in Grafana's *frontend* - `/api/ds/query` always returns
per-series frames, so you cannot verify it from the API and the usual result is
a silently empty panel. Used **`bargauge`** instead: it consumes instant series
directly and names each bar from `legendFormat`, so what is validated against
Prometheus is what renders. Same reasoning applies to any "one value per label
combo" panel.

Validation worth repeating for any new dashboard here: extract every panel
`expr`, substitute `$vars`, and POST each to `/api/v1/query` - catches typos and
wrong monkit field names before shipping. Field convention: monkit **counters**
expose `{field="value"}` (plus high/low); **observations** (IntVal/FloatVal/
DurationVal) expose `count/max/min/ravg/recent/sum`. `edge_download_success` and
`relay_circuit_rejections` are counters; `relay_circuits_active`,
`edge_download_speed_bps`, `relay_worker_rtt_*` are observations.

**80 MiB concurrency-8 false all-fail (2026-08-03).** The wrapper passed
`-timeout 12m`, but sdk-benchmark used that only for its outer context and did
not pass it to `uplinksdk.WithDownloadTimeout`, so every SDK download retained
the 2-minute default. Eight 80 MiB downloads need about 132-138s on the dev
relay: all eight deterministically expired after 120s despite healthy uploads.
Fixed by retaining the parsed timeout in benchmark config and passing it into
the SDK. The wrapper now always invokes cached `go build` rather than silently
reusing an old binary, and puts `GOTMPDIR` under `$XDG_CACHE_HOME`/`$HOME/.cache`
because the root filesystem may be full while `/home` has space.

Reproduction evidence: same seeded corpus hash-passed at concurrency 1 (13s),
2 (4/4), and 4 (4/4); original concurrency 8 was 0/8 at 125s. After timeout
wiring, concurrency 8 hash-passed 8/8 in 139s. One intervening run still failed
8/8 from transient remote stream resets, and the green run logged 427 failed
redundant piece attempts before collecting 29 shares per segment. Relay telemetry
showed no new circuit rejections and only ~298/10000 active circuits, so residual
stream-reset flakiness is real but separate from the deterministic timeout bug.
`internal/piecedownload.Manager.Fetch` now debug-logs each failed piece with its
piece number and error, which preserves timing needed for future diagnosis.
Detailed green-run counts: 960 piece attempts started (8 downloads x 2
prefetched segments x 60 committed pieces), 427 actual failed attempts, all
`Unavailable` remote stream resets. The error text contains `stream reset
(remote)` twice, so raw substring counting reports 854 but there were 427
failed attempts. The first major correlated burst was 200 failures around
11.5-12.1s after launch; failures continued in waves until ~85s, so describing
all 427 as occurring at 11s is inaccurate. The unaccounted attempts were
long-tail work canceled/ignored after enough shares were collected. Healthy
long-tail cancellation is distinct from the 427 resets: those were observed by
Manager before it reached the 29-share threshold. Correlated same-millisecond
bursts suggest a shared worker/relay transport boundary, but the current failure
log lacks worker ID/circuit identity, so it cannot yet attribute which boundary
issued the reset.

**Adaptive 29+6 scheduling + serial segment default (2026-08-03).** Replaced
piecedownload's refill-on-every-completion semaphore with Storj's K-of-N
remaining-successes window: `DefaultLongTail=6`, so 29-of-60 starts 35; failure
pulls one reserve immediately; success shrinks the target and does not refill;
29 successes cancel the remaining six. Explicit lower
`WithDownloadConcurrency` caps still work. Also changed
`DefaultSegmentPrefetch` 2 -> 1 (explicit `WithSegmentPrefetch(2)` remains),
because two eager segment opens multiplied the piece fan-out without creating
bandwidth on the shared relay.

RED/GREEN test `TestFetch_UsesRequiredPlusLongTailWindow` proves initial 35,
failure replacement, no success replacement, and cancellation at 29; default
prefetch test now pins 1. `go test -race ./internal/piecedownload` and relevant
SDK packages pass. Live 8x80MiB result with both changes: **8/8 hash OK, 67s,
9.55 aggregate MiB/s, zero piece failures/resets**. It started exactly 560
piece attempts over the whole run (35 x 16 segments), but only one segment per
download was active, so peak initial pressure was 280 vs old 960. All 16
segments completed exactly 29 pieces (464 total), and 96 long-tail attempts
(6 x 16) were canceled. Long-tail35 alone with prefetch2 was only 6/8 and 153s,
showing segment fan-out was the decisive multiplier. Full `go test ./...` has
one unrelated existing failure: `internal/vo.TestPieceIDScanNullAndEmpty`.

**Post-fix speed ceiling.** Fresh 1x80MiB baseline: 11.58s end-to-end,
SDK wall 10.05s / 8.0 MiB/s, piece download 7.12s, decrypt 1.26s, RS decode
0.72s, zero failures (70 starts = 35 x 2 segments, 58 completions = 29 x 2).
At 8-way, each SDK wall was ~58-62s and piece download ~55-58s while
decrypt+decode stayed ~2.0-2.4s; network/piece retrieval is therefore ~94-96%
of wall and CPU is not the primary aggregate bottleneck. Aggregate app
throughput was 9.55 MiB/s while relay telemetry peaked around 13.9 MB/s relayed
payload (12-14.6 MB/s ingress/egress). Biggest remaining speed lever is bypassing
or scaling the shared relay/direct worker reachability; streaming RS decode and
a client-global work budget would mainly improve TTFB/memory and safely permit
conditional segment overlap. A/B long-tail 32/35 may reduce partial overfetch,
but trades away straggler tolerance.

**Pre-existing blocker in the monitoring stack:** `monitoring/.env` is missing
`WORKER_METRICS_BEARER_TOKEN`, which `docker-compose.yml`'s caddy service
requires with `${...:?}`, so *any* `docker compose` command in that directory
fails to interpolate. Pushgateway had to be started with a bare `docker run`
(name `monitoring-pushgateway-1`, network `monitoring_default`, alias
`pushgateway`). Add the var to `.env`, then `docker rm -f
monitoring-pushgateway-1` before `docker compose up -d` so compose can manage it.

**Client-wide piece budget + relay buffer decision (2026-08-03).** The SDK now
has one reader-lifetime `piecedownload.Limiter` per Client, configurable with
`WithGlobalDownloadConcurrency`; the non-disableable default is 280, matching
8 concurrent downloads x one active segment x the 29+6 adaptive window. A token
is acquired before `GetPiece` and held until its returned reader closes, so it
limits actual streams rather than only stream setup. Focused RED/GREEN tests
cover blocking, release-on-close, and context cancellation; relevant race tests
pass. Live validation remained hash-correct at 8/8, but the remote path had
degraded: 169s / 3.79 MiB/s, 199 remote piece failures, 752 attempts versus the
clean baseline's 560; a following 1x control was only 2.58 MiB/s versus the
earlier 8.0 MiB/s, so the speed difference is not evidence of limiter queuing.

Kept relay `resources.buffer-size` at 8192. libp2p defaults to 2048; at the
observed 13.9 MB/s peak, 8 KiB is only about 1,700 aggregate copy iterations/s.
Each circuit allocates two buffers, so 64 KiB would reserve about 128 KiB per
circuit (~38 MiB at 300 active circuits) without CPU/syscall evidence that copy
buffering is the bottleneck. Do not raise it without a controlled 8 KiB/64 KiB
A/B including relay CPU, RSS, throughput, failures, and identical corpus/load.

Prometheus and Loki history was reset on 2026-08-03 by deleting only
`monitoring_prometheus-data` and `monitoring_loki-data` and recreating those two
services. Both became ready; active remote-write/log-shipping clients immediately
started repopulating fresh data. Pushgateway was intentionally not cleared.

**Post-relay-reload validation (2026-08-03).** Repeating the exact 8x80MiB SDK
workload after the user reloaded the relay produced **8/8 hash OK, 75s, 8.53
aggregate MiB/s, zero piece failures**. It returned to the ideal scheduler counts:
560 starts (35 x 16 segments), 464 completions (29 x 16), and 96 canceled long
tails. Post-run relay status showed ~15.4 MB/s relayed payload, 150 active
circuits, 150 reservations, and only one additional circuit rejection across
the sampled before/after counters (18 -> 19). This is close to the prior clean
67s / 9.55 MiB/s run and confirms the intervening 169s / 199-failure result was
relay-state variability, not the client-global limiter. `/stats.json` does not
expose the effective relay BufferSize, so this run alone cannot attribute the
recovery to a buffer setting unless the deployed config is recorded separately.

**Relay degradation root cause + global-cap ladder (2026-08-03).** The default
limited-connection pool path claimed relayed connections were single-use, but on
stream completion it returned them to cache. Their `Unblocked` channel is
deliberately never closed, so they could never be reused; persistent clients
therefore retained an idle relay circuit per completed piece until cache eviction.
Live evidence after a relay restart: active circuits climbed to 516 while payload
fell to ~0.2 MB/s. Fixed both mirrored `internal/grpcutil/pool/conn.go` copies to
close reuse-off limited connections when their stream context ends (and close them
on NewStream failure), with RED/GREEN race-tested regression coverage. Direct and
explicit bounded-reuse paths are unchanged. A three-point run using the fixed
local SDK raised relay cumulative circuits by 628 but active circuits by only 18
during ongoing traffic, rather than retaining hundreds.

The benchmark now exposes SDK `-global-download-concurrency` and wrapper
`-G "140 210 280"`; Pushgateway grouping includes concurrency and global cap so
POSTing later ladder points no longer replaces earlier points. Exact 8x80MiB
results were: cap 140 = 7.80 MiB/s / 82s / 3 recovered piece failures; cap 210 =
8.42 MiB/s / 76s / 5 recovered failures; cap 280 = 9.41 MiB/s / 68s / zero piece
failures. Keep 280 for the current healthy relay.

Grafana discrepancy: `dl_bench_throughput_mibps` is aggregate across all eight
SDK downloads, but `edge_download_speed_bps` is bytes/wall for one edge request;
the SDK benchmark arm bypasses edge entirely. After monitoring reset, live edge
metrics showed process-lifetime `ravg` ~1.0 MB/s and max watermark ~5.2 MB/s,
consistent with each 80MiB request taking ~61-65s while aggregate SDK throughput
was 9.4 MiB/s. The dashboard descriptions now state this explicitly. A true edge
8x80MiB control failed 0/8 with exact 50s 504s; Loki showed `cannot reserve
outbound stream: resource limit exceeded`, and Prometheus identified deployed edge
commit `c0ed626`. That deployment still has the stale-circuit bug (534 active
circuits observed) and must be rebuilt/restarted with the fixed go-sdk before
retesting; the local SDK benchmark uses unlimited rcmgr limits and did not hit the
edge process's exhausted outbound stream scope.

**No configured 8 MiB/s rate cap (2026-08-03).** Audited the edge, SDK, relay,
and worker download path after the apparent single-download ceiling. There is no
byte-rate throttle: edge only sets piece concurrency and timeout; SDK's 280 global
budget and libp2p rcmgr are stream/resource counts; relay BufferSize is copy chunk
size, MaxCircuits is a count, and per-circuit Data/Duration are 256MiB/30m limits;
worker download sends 64KiB frames in a tight read/send loop and its concurrent
request default is unlimited. Resource-manager exhaustion rejects streams rather
than shaping them to 8 MiB/s. The observed healthy results (one request ~8.0
MiB/s, eight requests ~9.4 MiB/s aggregate, relay payload ~15.8 MB/s) indicate an
empirical shared transport ceiling around the relay/client path, not an edge rate
setting. Isolating NIC vs relay CPU vs TCP/libp2p flow control requires host CPU,
NIC and TCP measurements during a size/concurrency ladder.

**Edge redeploy recheck still old/unfixed (2026-08-03).** After the user said a
new edge version was deployed, the exact 8x80MiB edge test still returned 0/8,
all HTTP 504 with 93-byte bodies at 50.17-50.19s (`dl-20260803-112905`). Loki
still showed `cannot reserve outbound stream: resource limit exceeded` plus dial
backoff, with segments collecting only 8-21 of 29 pieces. Relay active circuits
remained exactly 534 before and after, and Prometheus still reported edge build
commit `c0ed626`, instance/container ID `d77a8cf42670`. Therefore the public edge
was still the old process or was rebuilt without the uncommitted local go-sdk
pool fix. The fix is only in the dirty local go-sdk/depin worktrees until it is
committed/pushed/re-pinned or explicitly included in the deployed build.

**Confirmed new edge container, rcmgr still blocks 8-way (2026-08-03).** A later
retry saw new Prometheus instance/container `81279b19f236`, confirming replacement,
but `dl-20260803-113303` still returned 0/8, all 504 at 50.18-50.22s. New-instance
Loki logs show the edge host itself still fails to reserve outbound relay streams
with `resource limit exceeded` and dial backoff; segments reached only 13-22 of 29.
The new instance recorded eight edge errors and ~2.23 MB/s SDK fetch `ravg` for
transfers that progressed before timeout. This confirms the remaining blocker is
edge libp2p resource-manager capacity, independent of whether the circuit-close
fix is present. The SDK benchmark succeeds because it explicitly uses unlimited
resource limits; edge currently does not pass any `WithResourceLimits` option.

**Edge SDK resource-limit fix implemented (2026-08-03).** go-sdk now publicly
aliases `ResourceLimitConfig` and its internal copy gained the same Conn and
PeerDefault inbound/outbound stream overrides already present in depin. Edge
`Config` now exposes `ResourceLimits`, passes it through
`uplinksdk.WithResourceLimits`, and resolves unset system, per-connection, and
per-peer outbound stream caps to 512. This is bounded headroom above the SDK's
280 global piece budget rather than disabling rcmgr. Explicit operator values are
preserved. RED/GREEN tests pin default 512 and overrides; go-sdk root/p2putil/pool
and depin edgeserver race tests pass; a real edge binary builds and `run --help`
exposes all three `--resource-limits.max-*-outbound-streams` flags. Requires one
more edge rebuild/redeploy before the live 8x80MiB edge retry.

**Post-resource-limit deploy result (2026-08-03).** New edge container
`9d356109160c` removed the outbound rcmgr failure: the 8x80MiB test no longer
returned 504 or logged `cannot reserve outbound stream`. Instead all eight reached
segment 2 and transferred 67.45-71.54 MiB, then hit the SDK's 2-minute deadline
(`context deadline exceeded: sdk: stream segment 2: eestream: read completed
buffer`). Public responses were HTTP 200 with short bodies at 123.67-124.56s, so
the proxy still made an upstream abort look like clean EOF to curl; harness
correctly counted 0/8. Aggregate partial-byte rate was 4.42 MiB/s. Relay raw rate
during/after was ~8.2 MB/s, below the healthy direct-SDK run's ~15.8 MB/s. The run
opened 128 circuits (total 1731 -> 1859) and active rose 406 -> 534, remaining 534
after 15s, so deployed edge circuit cleanup still needs verification against the
exact embedded go-sdk build. Resource limits are fixed; current blockers are
throughput/deadline and detectable short-response propagation.

**Edge bottleneck trace (2026-08-03).** Historical Prometheus samples for the
11:43 edge run localize the slowdown to the deployed edge's relay connection
lifecycle, not worker storage or edge CPU. Across the ~124s run, edge rusage rose
by ~71 CPU-seconds (~0.57 cores average), heap peaked near 1.27GB and process RSS
near 1.66GB, so the process was not CPU-saturated. Worker process-lifetime
telemetry showed ~10.8MB/s average successful piece service and ~1.6ms average
TTFB. The edge, however, hit a `(*poolConn).NewStream` high-water mark of exactly
280, accumulated 128 relay circuits that remained active after the requests,
and completed 61 newly observed dials totaling 108.04s (~1.77s each). Relay
`recent` payload varied from 4.67 to 16.39MB/s (about 9.66MB/s average), proving
the relay could burst above the run's sustained rate rather than enforcing a
fixed cap. The retained-circuit signature matches the reuse-off stale-cache bug
already fixed in both local pool copies: the live edge was built without that
effective lifecycle fix or did not embed the expected go-sdk source. No new
instrumentation was required to distinguish this case. Redeploy an edge that
definitely embeds the fixed go-sdk, restart it to clear retained circuits, and
repeat the same 8x80MiB workload before pursuing buffer or worker changes.

**Live inactivity-timeout validation (2026-08-03).** After deploying a new edge
instance `578956cfcb99`, reran the exact 8x80MiB edge workload as
`dl-20260803-134148`: **8/8 succeeded**, every response was HTTP 200 with exactly
83,886,080 bytes, request durations were 137.15-141.41s, and aggregate throughput
was 4.51MiB/s. This crosses the old two-minute absolute deadline and therefore
validates that active progress now keeps the request alive. Edge telemetry
recorded eight successes, zero errors, and 136.96-141.23s request durations.
Relay active circuits rose 406 -> 534 (total 1859 -> 1987) and stayed at 534 after
20s. This was initially misclassified as failed gRPC pool cleanup. It actually
tracks libp2p peer circuits, which the persistent edge swarm may retain after the
per-piece gRPC/libp2p streams close; see the 2026-08-03 correction in
[[grpc-pool-reuse-relay-regression]]. The actionable speed cost is repeated
gRPC/TLS setup over those warm circuits, now targeted by the edge's conservative
bounded-reuse default of 2, pending redeploy and another 8x80MiB test.
