# Plan: Edge download-speed improvements (steps 4, 2, 3)

Date: 2026-07-29
Status: Draft (approved to save; not yet implemented)
Scope: improve edge-server download throughput for the k6 / HLS workload, under the constraint that many workers have no public IP (relay is mandatory; direct / dual-address is off the table).

## Context and root cause (verified)

Edge download is latency-bound, not bandwidth- or disk-bound.

Phase breakdown per download (edge-server1, production k6/HLS), total ~865 ms:
- piece_download: 357 ms (this is connection setup, ~4-5 round trips x 38 ms WAN relay RTT, 2 waves at concurrency 16)
- erasure_decode: 30 ms
- decrypt: 25 ms
- remainder ~363 ms: coord GetDownloadInfo + edge serve + TTFB

A worker serves a 16 KB GET piece in ~1 ms at ~15 MB/s, so neither the worker nor the disk is the bottleneck. The GET_AUDIT rate metric (256 B fixed payload) is a meaningless artifact and should be ignored.

Correction to an earlier hypothesis: the relay per-circuit data cap is NOT the limiter. It is 256 MB / 30 min (coord/relay/config.go, Limit.Data default 268435456, Duration default 30m); a failing download moved about 48 KB, roughly 5000x under the cap. Raising the relay data cap does nothing for RTT.

Already handled:
- Download concurrency set to unlimited (collapses the 2 fetch waves into 1; ~357 ms to ~180 ms per segment).
- go-sdk piece download already does long-tail cancellation (fetch, use first-k, cancel stragglers), so straggler / overfetch handling is in place when concurrency > required-k.
- The reset failure ("fetched 3 pieces, need 29", GOAWAY 4101) is fixed by single-use relay connections (opt 1) plus dead-conn eviction (opt 2).
- CDN cache headers exist on the edge (client-facing only).

## Workstream 4 - Reproduce and pin the exact reset limit (prerequisite for WS2)

Why: the real limit that breaks connection reuse over a relay circuit is unconfirmed. Candidates: worker libp2p resource-manager (rcmgr) scope caps, gRPC server MaxConcurrentStreams, or HTTP/2 behavior over a single libp2p stream. One grpc.ClientConn equals one libp2p stream (HTTP/2 sub-streams inside it), so the libp2p per-stream cap may not even be the download trigger. Pin it before raising anything.

Steps:
1. Stand up a minimal relayed path: 1 coord + 1 relay + 1 worker forced relay-only + 1 client/edge. Local stack via dev/coord-dev.sh + identities/relay1 + a single relay-only worker, trust cache wired.
2. Enable worker-side rcmgr tracing (rcmgr.WithTraceReporter, or GOLOG_LOG_LEVEL="rcmgr=debug", or a temporary trace file) so "resource limit exceeded" rejections are visible with the scope that fired (System / Peer / Conn / Stream).
3. Drive load with a harness that multiplexes N concurrent piece streams over ONE reused relayed connection (temporarily allow reuse of limited conns, or use the pre-fix broken-reuse build). Ramp N = 1, 2, 4, 8, 16, 29.
4. Record the N at which the worker resets (GOAWAY 4101) and the exact rcmgr scope/limit in the trace.
5. Also capture whether the gRPC server default MaxConcurrentStreams plays a role, and confirm the one-libp2p-stream assumption via connection counts.

Output: the concrete limiting scope and numeric threshold (the safe streams-per-conn value M). Deliverable is a short findings note, no code shipped.

Effort: about 0.5 to 1 day (mostly stack setup). Local-stack caveats from prior work: stale coord-url in dev scripts, piece_key.json format, relay uses insecure transport.

## Workstream 2 - Bounded connection reuse over the relay

Why: reuse amortizes the ~4-5 round-trip setup (x38 ms) across all pieces and all HLS segments. This is the core RTT fix for the non-cached path. Depends on WS4's threshold M.

### 2a. Worker - lift the per-connection stream cap
- The current knob ResourceLimits.MaxInboundStreams overrides System scope only, which is insufficient; the per-connection cap is what fires.
- Options: set worker ResourceLimits.Unlimited=true (simplest; the code comment already recommends it for heavy fleets, accepting OOM/FD risk since workers are ours), OR extend internal/p2putil/resourcelimit.go systemOverrides() to also raise ConnScope/PeerScope StreamsInbound to a configured value (cleaner, targeted). Recommendation: the targeted knob over blanket Unlimited.
- Files: internal/p2putil/resourcelimit.go, worker/server/server.go (config plumbing), fleet config default.

### 2b. Client / edge - bounded reuse of limited connections
- Relax the opt-1 gate: instead of "limited implies single-use", allow limited conns to be reused but with capped concurrency per conn = M (from WS4) plus a small warm pool of K conns per worker (round-robin). Amortizes setup, spreads load across a few circuits, keeps blast radius small, stays under the reset threshold.
- Implementation in internal/grpcutil: add a per-conn in-flight-stream counter; the pool hands out a limited conn only while its stream count is below M, otherwise opens/gets another up to KeyCapacity. Keep opt-2 Broken() eviction for resilience.
- Add keepalive to grpcconn.New's ClientConn (the pool path currently has none; the upload path sets it at piecestore.go:82). Detects dead/hung reused conns, warms them, and closes the earlier goroutine-leak concern.
- Files: internal/grpcutil/{pool/conn.go, pool/pool.go, grpcconn/conn.go} plus the go-sdk siblings, and edgeserver/server.go for concurrency/pool config.

### 2c. Relay - headroom for long reused circuits
- With reuse, one circuit per worker per session accumulates bytes and time. The current 256 MB / 30 min cap is generous, but a long VOD/HLS session could approach it. Plan: monitor relay_circuit_lifetime and bytes; raise Limit.Data / Duration or rotate a circuit before the cap. Config-only, coord/relay/config.go.

Risks: reuse concentrates load onto fewer circuits (larger blast radius than single-use), mitigated by the K-conn pool plus Broken() eviction plus keepalive. Must not exceed WS4's M.

Validation: k6 before/after; watch edge_sdk_phase_piece_download, connection_from_cache vs connection_dialed, relay_circuit_rejections, and confirm zero GOAWAY resets.

Effort: about 2 to 3 days after WS4.

## Workstream 3 - Parallel / prefetch segments (go-sdk)

Why: go-sdk/download.go:269 and :424 loop segments strictly serially (Open(seg) then io.Copy then next). Segment N+1's piece fetch does not start until N is fully streamed, so multi-segment (VOD/large) objects pay each segment's fetch latency back to back.

Design:
- Add WithSegmentPrefetch(depth), default 2; depth 1 equals current behavior.
- Replace the serial loop with a bounded read-ahead pipeline: open up to depth segments ahead concurrently (each segmentdownload.Open kicks off its piece fetches), hold their io.ReadCloser handles in an ordered ring, and io.Copy to the output in segment order. This overlaps N+1 fetch with N serving.
- Apply to both the full loop and the range loop; preserve ordering, propagate errors, cancel look-ahead on failure, and cap memory (depth x segment size).
- Files: go-sdk/download.go (both loops), go-sdk/options.go (new option), possibly a small ordered-prefetch helper.

Caveat: helps large multi-segment objects; little effect on small single-segment HLS chunks (the ~480 KB samples observed are single-segment). Sequence after WS2 unless VOD is a priority.

Risks: memory growth (bounded by depth), ordering bugs, error propagation from look-ahead. Cover with unit tests plus a testplanet e2e (extend TestRangedDownload).

Effort: about 1 to 2 days, isolated to go-sdk.

## Sequencing and measurement

1. WS4 (pin limit) -> 2. WS2 (worker cap + bounded reuse + keepalive) -> 3. WS3 (segment prefetch; parallelizable with WS2).

- Baseline now, re-measure after each change via k6 plus the edge phase metrics.
- Target: piece_download from ~180 ms to ~40-60 ms via reuse; for multi-segment objects, wall-clock approaching max(segment) rather than sum(segments).
- All changes touch both depin and the go-sdk sibling; go-sdk needs a new pseudo-version pin in depin go.mod after each merge.

## Related

- Not viable here: dual-address / direct edge-to-worker (many workers lack public IP).
- Bigger separate win for repeated HLS content: an edge-origin LRU cache (deferred by user for now; would skip coord + relay + worker on cache hits).
- Prior context: relay hairpin throughput diagnosis, relay observability metrics, edge download timeout fix.
