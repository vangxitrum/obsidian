---
type: fact
tags: [grpcutil, pool, relay, libp2p, download, regression, edgeserver]
created: 2026-07-24
agent: main
---

Commit `79ae347 fix(grpcutil): enable connection pool reuse for concurrent streams`
broke edge/SDK downloads over the relay: `edgeserver ... download failed: sdk: open
segment 1: piecedownload: fetched 3 pieces, need 29: workerclient: open download
stream: rpc error ... failed to write client preface: connection closed (remote):
code: 0x1005 ... remote sent go away, code: 4101`.

**Root cause:** the commit made two changes — `grpcconn.New` closes `unblockedChan`
immediately (was never closed → `cache.Take` skipped every entry → 100% cache miss,
so reuse was effectively OFF), and `pool.NewStream` returns the conn to the pool
right after opening the stream. Net effect: many concurrent piece-download gRPC
streams to one worker now MULTIPLEX over a single grpc.ClientConn = a single libp2p
STREAM (`internal/grpcutil/connector/p2p/conn.go` Dial wraps one `host.NewStream`
`WithAllowLimitedConn("relay-dial")`). NAT'd workers are reachable only via a
circuit-relay-v2 LIMITED connection (hard data/time cap). Shared budget dies after
~3 pieces → relay resets circuit (code 4101/0x1005) → all HTTP/2 substreams reset →
gRPC tries to reconnect but grpcconn's dialer returns a FIXED single-use libp2p
stream (can't re-dial) → "failed to write client preface" → pooled ClientConn
poisoned and kept getting reused. Before the commit each piece dialed its own libp2p
stream, so downloads worked.

**Prometheus confirm:** `connection_from_cache{role="edge-server1"}=252495` on a
fresh instance (deploy ~10:5x, err 10:51Z), was ~0 before by design;
`connection_dialed≈194/worker`; transport relayed (`relay_circuits_active~11700`).
Same fix was applied to go-sdk's sibling `internal/grpcutil/pool` too (edge uses go-sdk).

**Secondary bug:** `pool/conn.go` NewStream calls `c.pool.put(c.pk, pv)` BEFORE the
err check → a just-failed conn is returned and reused; `Stale()` only checks explicit
Close()/MaxLifetime, never grpc connectivity state → dead conns never evicted.

**Fix IMPLEMENTED 2026-07-24 (depin, uncommitted; go-sdk port in progress):**
- Opt1 (reuse only direct): `LibP2PConn.Limited()` = `stream.Conn().Stat().Limited`;
  `connector.isLimited()` unwraps net.Conn chain via `NetConn()`;
  `tlsConnWrapper.Limited()`; `grpcconn.New` sets `reusable=!Limited` (duck-typed
  optional iface), only `close(unblockedChan)` when reusable; adds `Conn.Reusable()`.
  `pool.poolConn.NewStream` branches: direct=put back immediately (multiplex),
  limited=hold checked-out for stream lifetime (`go func(){<-stream.Context().Done();
  put}()`) so each relayed piece gets its own circuit AND cache eviction can't close
  an in-use limited conn.
- Opt2 (evict dead): `grpcconn.Conn.Broken()` = ClientConn.GetState() in
  {Shutdown,TransientFailure}; pool `Stale()` returns true if Broken() -> Cache.Take
  closes+skips it, dials fresh. (`RawConn` iface gains Reusable()+Broken().)
- Tests: pool/conn_test.go (limited-not-shared, limited-not-reused-sequentially,
  broken-evicted) + grpcconn/conn_test.go (LimitedConnNotReusable, DirectConnReusable);
  proven FAIL pre-fix (simulated revert) / PASS post-fix. grpcutil all green;
  worker/edge/coord/pkg/internal build clean (only pre-existing cmd/uplink drift fails).
- Opt3 (relay reservations/limits): NO CODE - relay already supports it. Per-circuit
  cap = `coord/relay/peer.go` `config.Limit` -> `WithLimit`/`WithInfiniteLimits`;
  reservations via `config.Resources.MaxReservations`. A reserved circuit is STILL
  Limited=true so it does NOT re-enable pool reuse (opt1 gate stays). Client-side
  `connector.Reserved()` dial-option is DEAD CODE (never wired; download path passes
  empty dial.DialOptions{}). Recommendation = operator config: keep finite relay
  Limit.Data for DoS safety + rely on opt1/2; raise Limit.Data only if a single-piece
  fetch exceeds it.

Related: [[relay-connected-peers-vs-reservations]], [[edge-download-timeout-fix]],
[[relay-domain-locator-egress]].


**2026-07-26 CORRECTION + edge download-speed investigation:**
- WRONG earlier claim: reset was the relay per-circuit DATA cap. FALSE - `coord/relay/config.go` `Limit.Data default=268435456` (256MB), `Duration default=30m`; a failing download moved ~48KB, 5000x under. Raising relay cap does NOT help RTT.
- REAL reset trigger (still needs live confirm): worker libp2p rcmgr caps inbound streams under multiplexing (documented at `worker/server/server.go:150-156` for uploads: autoscaled rcmgr rejects inbound streams -> "resource limit exceeded"). NUANCE: one grpc.ClientConn = ONE libp2p stream (HTTP/2 sub-streams inside), so libp2p per-stream cap may NOT be the DOWNLOAD trigger; candidates = rcmgr scope caps / gRPC MaxConcurrentStreams / HTTP2-over-one-stream. `ResourceLimits.MaxInboundStreams` only overrides SYSTEM scope, NOT per-connection - only `Unlimited=true` lifts per-conn caps.
- Edge download is LATENCY-bound, not bandwidth/disk. Phase breakdown (edge-server1, prod k6/HLS): total 865ms = piece_download 357ms (setup RTT, ~4-5 round trips x 38ms WAN relay RTT, 2 waves at concurrency 16) + erasure_decode 30ms + decrypt 25ms + ~363ms remainder (coord GetDownloadInfo + serve + TTFB). Worker serves a GET piece (16KB) at ~15MB/s (~1ms) - NOT the bottleneck. GET_AUDIT rate (256B fixed) is a meaningless artifact.
- Fixes user applied: download concurrency -> unlimited (all k pieces one wave; ~357->~180ms). go-sdk piecedownload ALREADY does long-tail cancel (fetch, first-k-wins, cancel stragglers) so overfetch/straggler handled when concurrency>k.
- Remaining opt levers (ranked): (1) EDGE-ORIGIN CACHE (LRU) - none today, only CDN Cache-Control headers; biggest HLS/k6 win (skip coord+relay+worker on repeat). (2) bounded conn reuse over relay (amortize setup RTT) - needs worker rcmgr cap raised + client gate relaxed. (3) parallelize/prefetch segments (go-sdk download.go:269,424 loops segments SERIALLY). (4) multiple relays + geo-placement (single relay = RTT+bw chokepoint). (5) cache coord manifest/order-limits per object. (6) pipeline decode w/ piece arrival. (7) lower RS k. (8) confirm TLS1.3 on piece dial. (9) keepalive+warm libp2p conns (also fixes hung-conn goroutine leak). (10) relay BufferSize 8192.
- go-sdk fix f9cbaee committed (single-use relay conns, opt1+2) by tuan-be; depin opt1+2 uncommitted; keepalive-on-grpcconn.New proposed (pool path has NO keepalive; upload path does at piecestore.go:82) - not yet done.

**2026-08-03 circuit-lifecycle correction + conservative speed setting:**
`relay_circuits_active` counts libp2p peer circuits, not the per-piece gRPC
connections/streams managed by `internal/grpcutil/pool`. A successful piece path
does call `downloadStreamReader.Close()`, which cancels the gRPC stream context;
closing its `grpcconn.Conn` closes the libp2p stream but the swarm may correctly
retain the shared peer circuit. Therefore a persistent edge adding roughly one
circuit per contacted relayed worker is not proof that the pool's
`stream.Context().Done()` cleanup failed, and forcibly closing peers would reset
other concurrent downloads.

The measured speed cost is repeated gRPC/TLS setup over those warm peer circuits
(~1.77s per observed edge dial). The bounded-reuse implementation already targets
that layer. Edge now defaults `server.relay-reuse-max-streams` to **2**: enough to
halve setup churn, deliberately far below the old unbounded multiplexing behavior.
Operators can set 0 for single-use. A config-tag RED/GREEN test pins the default;
SDK pool and edge race tests pass, and built `edgeserver run --help` reports
`default 2`. Live 8x80MiB validation is still required after redeployment.
