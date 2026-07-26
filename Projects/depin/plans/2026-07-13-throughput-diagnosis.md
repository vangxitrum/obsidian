# Diagnose low upload/download throughput (client+workers local, coord+relay remote/WAN)

## Context

Upload and download speeds are too low under a load of ~100 concurrent runs where the
client and storage workers run **locally** but the coordinator + relay run on a
**remote (WAN) machine**. We need to locate the bottleneck empirically before changing
any behavior. Static code tracing already surfaced two strong suspects; this plan adds
the missing instrumentation and a measurement runbook to *quantify* their contribution,
then hands back a ranked attribution report. **Scope is diagnose/measure only - no
throughput behavior changes in this phase.** The chosen fix direction for the next phase
is the "best-of-both" dual-address topology fix (documented at the end, not implemented here).

### Suspected bottlenecks (from source tracing, to be confirmed by measurement)

1. **Relay hairpin (WAN) - prime suspect.** Workers force relay-only reachability:
   `worker/server/server.go:201` `ForceReachabilityPrivate()` + circuit-only advertised
   address, gated on `!HavePublicAddress` (`worker/peer.go:200`). Coord stamps that single
   circuit address into every order limit (`coord/order/signer.go:216`). So local-client to
   local-worker piece bytes cross the WAN **twice** through the remote relay
   (`coord/relay/peer.go`, byte counter `coord/relay/tracer.go:29`). Local bandwidth is
   irrelevant; throughput is capped by WAN links + relay per-circuit limits
   (`coord/relay/config.go`: `BufferSize=8192`, per-conn `Data=256MiB`, `Duration=30m`).
   Hole-punching is wired (`server.go:166`) but never upgrades here (client+worker share one
   NAT; piece stream is bound to the relayed conn).
2. **Serial upload.** `go-sdk/internal/pieceupload/manager.go:130` uploads pieces **one at a
   time, in order**, needing `optimalThreshold=50` successes and failing if <50
   (`manager.go:167`). Over WAN that is ~50 sequential stream round-trips per 64 MiB segment.
   The deprecated `depin/uplink/.../upload.go:378` path was parallel; the shipped go-sdk path
   regressed to serial. Amplifies with the relay hairpin.
3. **Tiny upload chunk frame.** `go-sdk/internal/workerclient/client.go:24`
   `uploadBufferSize = 2 KiB` -> ~1130 gRPC `Send()` frames per 2.26 MiB piece (framing/CPU overhead).
4. **Low download concurrency.** `go-sdk/internal/piecedownload/manager.go:19`
   `DefaultConcurrency=4`, not overridden by the edge server (no `WithDownloadConcurrency`).
5. **Sequential segments** on both paths (`go-sdk/upload.go:171`, `download.go:173`).

### The instrumentation gap that blocks diagnosis

The **upload path has no exported throughput metric** - `worker/piecestore/endpoint.go:171-172`
only logs `upload_size`/`upload_rate` as zap fields. Download is fully instrumented
(`endpoint.go:643-708`, TTFB at `backend.go:340`). We cannot attribute upload time without
closing this gap first.

## Plan

### Step 1 - Add upload throughput instrumentation (measurement only, no behavior change)

Mirror the existing download metrics on the upload branch of the worker piecestore endpoint.

- File: `depin/worker/piecestore/endpoint.go`. In the upload handler (around `:171`), add monkit
  meters/counters paralleling the download ones at `:643-708`:
  `upload_started_count`, `upload_success_count`, `upload_failure_count`,
  `upload_success_byte_meter`, `upload_success_rate_bytes_per_sec`,
  `upload_duration_ns`, and `upload_time_to_first_byte_read` (mirror `backend.go:340`).
  Reuse the exact monkit patterns already in the download block so series names/shape stay consistent.
- Confirm the SDK-side per-phase timers already cover upload: `edge_sdk_phase_piece_upload`
  and `erasure_encode` (`go-sdk/internal/segmentupload/segment.go:211,296`, sink at
  `go-sdk/upload.go:240`, surfaced as `edge_sdk_phase_*` in `edgeserver/metrics.go:80`). No change
  expected here; just verify they fire under the benchmark.
- These are additive counters/meters only. No transfer logic is touched.

### Step 2 - Establish the metric-collection path for the run

- Point telemetry (`depin/pkg/telemetry/config.go`) at the existing monitoring stack, or scrape
  the private debug endpoints, to capture during the benchmark:
  - Edge: `edge_download_speed_bps`, `edge_download_duration`, `edge_sdk_fetch_speed_bps`,
    all `edge_sdk_phase_*` (`edgeserver/metrics.go`).
  - Worker download: `download_success_rate_bytes_per_sec`, `download_time_to_first_byte_*`
    (`endpoint.go:703-705,793`, `backend.go:340`) and the new `upload_*` meters from Step 1.
  - Relay: `bytesRelayed` (`coord/relay/tracer.go:29`) - confirms bytes are hairpinning the WAN.
  - P2P health: `p2p_relayed_only_peers` vs `p2p_direct_peers`, `p2p_direct_upgrades_total`
    (`internal/p2pmonitor/observer.go:180-205`) - confirms nothing upgraded to direct.
  - Dial pressure: `p2p_dial_backoff` meter (`internal/grpcutil/connector/dial.go:31`).
- Set `DropQuantiles=false` (`telemetry/config.go:58`) for this run so latency percentiles
  (r50/r90/r99) survive on the `mon.Task`/`DurationVal` series.

### Step 3 - Run the benchmark matrix and isolation experiments

Use the existing harness - no new benchmark code needed.

- **Over-the-wire, real WAN coord** (baseline): `dev/worker1/benchmark.sh` (wraps
  `go-sdk/cmd/sdk-benchmark`). Sweep sizes x worker-concurrency, split upload vs download rows:
  - `MODE=quick`, then `SIZES="1MiB 30MiB 256MiB" WORKERS="1 8 15"` for the knee.
  - `UPLOAD_ONLY=1` to isolate upload cost.
  - Point `UPLINK_SDK_COORD_URL` at the **remote** coord (the script currently targets a LAN
    `10.0.0.67`; use the real WAN coord URL for the representative measurement).
- **CPU-only, no network** (`go-sdk/cmd/pipeline-benchmark`, `-mode reed-solomon -size 256MiB`):
  isolates encrypt/encode/decode/decrypt. If CPU time is small vs the wire time, it rules
  encoding out and points at network/relay.
- **Topology A/B (the decisive experiment):** run the same sweep with coord+relay reachable
  over **LAN** vs **WAN**. The delta is the WAN/relay-hairpin cost. If LAN is fast and WAN is
  slow, the relay hairpin is confirmed as dominant.
- **Direct-vs-relayed spot check (reversible):** on a single throwaway worker set
  `Server.HavePublicAddress=true` + a reachable `Contact.ExternalAddress`
  (`worker/server/server.go:40`, `worker/contact/service.go:38`) so it advertises a **direct**
  address, and measure that worker's piece throughput vs a relay-only worker. Quantifies the
  ceiling a topology fix would unlock. This is a config toggle on a test worker, not a code change.

### Step 4 - Attribution report

Combine the numbers into a ranked breakdown per direction (upload vs download):

- Wire time vs CPU time (Step 3 CPU-only vs over-the-wire).
- WAN/relay contribution (Step 3 LAN vs WAN delta; corroborate with `bytesRelayed` and
  `p2p_relayed_only_peers`).
- Serial-upload contribution (per-segment `edge_sdk_phase_piece_upload` total vs single-piece
  time x count; a ~50x serial factor is the tell).
- Per-piece framing overhead (does 2 KiB chunk inflate CPU/`Send` counts materially).
- Download concurrency headroom (throughput vs the fixed 4-way fan-out).

Deliver a short written report ranking the confirmed bottlenecks with the measured evidence,
and a go/no-go recommendation for the Phase-2 fixes below.

## Recommended Phase-2 fixes (OUT OF SCOPE here - documented for the report)

Ordered by expected impact given a WAN topology:

1. **Best-of-both dual-address (chosen direction).** Stop collapsing the worker to a single
   circuit address: advertise **both** a directly-dialable address and the circuit address, and
   let the dialer race them (libp2p happy-eyeballs) so the local client goes direct while the
   remote coord still reaches via relay. Touches worker advertise/reachability
   (`worker/server/server.go:178-203`, `worker/peer.go:200-284`), the coord-stored address model
   (`coord/contact/selectedworker.go`, `coord/order/signer.go:216`), and the SDK dialer
   (`go-sdk/internal/grpcutil/connector/dial.go`). This removes the WAN hairpin for co-located
   client+worker - the biggest lever.
2. **Parallelize upload** (`go-sdk/internal/pieceupload/manager.go`): bounded-concurrency Phase 2
   with long-tail cancel at optimal, mirroring the deprecated parallel path
   (`depin/uplink/.../upload.go:378-409`) and the download manager's `sem` pattern.
3. **Bump upload chunk** 2 KiB -> 128-256 KiB (`go-sdk/internal/workerclient/client.go:24`).
4. **Raise + expose download concurrency** (`piecedownload/manager.go:19`) and wire
   `WithDownloadConcurrency` into `edgeserver/server.go`.
5. Segment-level pipelining on both paths.

## Verification (of this diagnostic phase)

- Build stays green after Step 1: `go build ./...` in `depin` (and `go-sdk` if touched).
- The new `upload_*` meters appear in the telemetry/scrape output during a benchmark run
  (grep the emitted series), mirroring the existing `download_*` meters.
- `dev/worker1/benchmark.sh MODE=quick` completes against the remote coord and produces
  `BenchmarkUpload`/`BenchmarkDownload` rows with the integrity check passing.
- The LAN-vs-WAN A/B in Step 3 yields a concrete delta, and `bytesRelayed` /
  `p2p_relayed_only_peers` confirm whether piece bytes traversed the relay.
- Output: the Step-4 attribution report with ranked, evidence-backed bottlenecks. No transfer
  behavior was changed in this phase.
