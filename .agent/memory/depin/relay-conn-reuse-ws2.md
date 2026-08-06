---
type: decision
tags: [go-sdk, grpcutil, pool, relay, libp2p, rcmgr, keepalive, download, edge, ws2, throughput]
created: 2026-07-29
agent: main
---

**WS2 of the edge download-speed plan: bounded reuse of relayed worker
connections + worker rcmgr stream-cap knobs + keepalive.** Built GATED / OFF BY
DEFAULT (user chose "build blind, M configurable, validate later"). Depends on
WS4 (unmeasured relay reset threshold M) - the enable knob stays 0 until WS4 is
run live. UNCOMMITTED both repos. Builds off [[grpc-pool-reuse-relay-regression]]
(the opt1 single-use fix this deliberately makes relaxable) and
[[gosdk-segment-prefetch]] (WS3, same plan).

## Why default-off matters
Relaxing "limited(relay) conn => single-use" is exactly the change that caused
the GOAWAY 4101 resets (commit 297433a fixed it). So the reuse path is inert
unless an operator sets a positive cap. With the cap 0 the code paths are
byte-identical to today - proven by the pre-existing pool regression guards
(limited-not-shared / limited-not-reused-sequentially / broken-evicted) still
passing unchanged.

## WS2a - worker rcmgr per-connection stream cap (depin `internal/p2putil/resourcelimit.go`)
- The System scope override alone does NOT cap per-connection multiplexing; the
  Conn scope does. Added knobs: `MaxConn{In,Out}boundStreams`,
  `MaxPeer{In,Out}boundStreams` (0=autoscale inherit, -1=unlimited, +=cap).
- `systemOverrides()` renamed `partialOverrides()`; now also sets
  `PartialLimitConfig.Conn` + `.PeerDefault` StreamsInbound/Outbound (verified
  field names in rcmgr v0.46.0 limit_defaults.go: `Conn`, `PeerDefault`, `Stream`
  are all `ResourceLimits`). Auto-exposed on every role's config via cfgstruct
  (worker/relay/client all embed ResourceLimitConfig). Default 0 => unchanged.
- To ENABLE reuse safely the worker's `MaxConnInboundStreams` must be raised >= M
  before the client cap M is set, else the relay/worker rejects the multiplexed
  streams. Fleet default left 0.

## WS2b-keepalive - `grpcconn.New` (BOTH repos, unconditional, safe)
Added `grpc.WithKeepaliveParams(Time:10s, Timeout:3s, PermitWithoutStream:true)`
to the pooled `grpc.NewClient` (mirrors the upload path at
pkg/workerclient/piecestore.go:82; pool path had none). Detects hung/dead reused
conns instead of blocking a stream until the download timeout. GOTCHA: goimports
strips a just-added-but-not-yet-used import between edits - add the usage first
(or in the same edit) so goimports ADDS the import rather than removing it.

## WS2b-reuse - gated bounded relay reuse (BOTH repos `internal/grpcutil/pool/`)
Design keeps the counter on `poolValue` (pool.go), NOT on the `RawConn`
interface - so grpcconn's interface is unchanged, no test-fake churn, no
grpcconn edit beyond keepalive.
- `pool.Options.LimitedReuseMaxStreams` (M; 0=off). `Pool.limitedReuseMax` mirror.
- `poolValue` gains `inFlight atomic.Int64` + `closeRequested atomic.Bool` (the
  same *poolValue pointer circulates across Take/Put, so the counter persists
  across a circuit's whole reuse life).
- cache Options closures (captured `opts`): **Unblocked** - direct conns share
  via their closed Unblocked() chan as before; a limited conn is shareable only
  when `M>0 && inFlight<M` (cache.Take skips it at the cap, so get() dials a
  fresh circuit). **Close** - if `M>0 && inFlight>0`, set closeRequested +
  return nil (DON'T hard-close an evicted-but-in-use circuit - that resets it
  mid-download); the last `release()` closes it once idle.
- `pool/conn.go NewStream` 3-way switch: direct=put-immediately (multiplex);
  limited+`limitedReuseEnabled()`=acquire+put-immediately+`go release on
  stream.Done`; limited+off=today's single-use (hold checked-out for stream
  life). `Pool.acquire/release/limitedReuseEnabled` helpers.
- Known scope limit (documented, not built): a HARD "K warm conns per worker"
  cap isn't naturally expressible in this cache (KeyCapacity bounds CACHED, not
  checked-out, conns). The idle-close-defer covers the eviction-of-in-use hazard
  instead. Fine for gated/experimental; revisit with WS4 data.

## WS2b-wire - enable knob
- go-sdk: `WithLimitedRelayReuse(maxStreams int)` client option -> `o.limitedRelayReuse`
  -> threaded through `dial.NewDefaultPooledDialer(conn, tls, limitedReuseMax)` ->
  `NewDefaultConnectionPool(m)` -> pool.Options. (Both dial ctors gained the int
  param; sole callers updated. NOT a Client field, so NOT in
  TestNewClient_CopiesAllOptionsFields.)
- edgeserver: `ServerConfig.RelayReuseMaxStreams` (default 0) -> passed as
  `uplinksdk.WithLimitedRelayReuse(...)` in server.go New().

## WS2c - relay headroom: NO CODE
`coord/relay/config.go` `Limit.Data` (256MB) / `Duration` (30m) are already
operator config knobs and already generous (a failing download moved ~48KB). Just
monitor relay_circuit_lifetime/bytes; raise or rotate if a long VOD session
nears the cap. Matches the opt3 "relay already supports it" finding.

## Tests (all green, -race)
- depin+go-sdk `pool/reuse_test.go` (mirrored verbatim; same package fakes):
  SharesUpToCap (M overlapping streams share 1 circuit, M+1 dials fresh, first
  circuit carries exactly M), SequentialReusesConn (M>0 reuses where default
  single-uses), EvictionDefersCloseUntilIdle (KeyCapacity 1 forces eviction of a
  saturated circuit; not closed while in-flight; closes after both streams
  cancel via require.Eventually).
- depin `p2putil/resourcelimit_test.go`: asserts Conn.StreamsInbound=512 /
  PeerDefault.StreamsInbound=1024 land on the right scopes.
- Existing pool regression guards unchanged + passing (default path identical).
- Full depin ./edgeserver + testplanet download e2e (TestSegmentPrefetchDownload
  / TestRangedDownload / TestEdgeRangedDownload) green against local go-sdk.

## Finalize (BLOCKING - depin does NOT build against the pinned go-sdk right now)
edgeserver now calls `uplinksdk.WithLimitedRelayReuse`, which only exists in the
uncommitted local go-sdk. `go build ./edgeserver/...` against the PINNED
`replace => gitlab.internal/...f9cbaeeb4a4e` FAILS ("undefined:
uplinksdk.WithLimitedRelayReuse"). Same finalize as [[gosdk-segment-prefetch]]:
user commits+pushes go-sdk (WS2 + WS3 together), then re-pin depin go.mod to the
new pseudo-version. Tested via temporary `replace aioz-depin/go-sdk => ../go-sdk`,
reverted after (Postgres :5445).

## Still deferred
WS4 = live relayed-stack measurement of the real per-circuit stream reset
threshold M (rcmgr trace, ramp N=1..29). Needed to pick a safe non-zero
RelayReuseMaxStreams + worker MaxConnInboundStreams. Until then the whole reuse
path ships inert. Related: [[grpc-pool-reuse-relay-regression]],
[[gosdk-segment-prefetch]], [[edge-download-timeout-fix]],
[[relay-connected-peers-vs-reservations]].
