---
type: plan
tags: [depin, libp2p, relay, nat, networking, throughput]
created: 2026-08-04
status: proposed
---
# Getting edge↔worker traffic off the relay

## Context

Every piece byte between the edge and a worker hairpins through the relay: 335
active circuits, ~2.8 GB relayed, `p2p_direct_upgrades_total == 0` across all
~150 workers — each has exactly one direct peer, the relay. Relay throughput
(~13.9-15.4 MB/s peak) is therefore the ceiling on aggregate download speed.

Two independent paths produce a direct connection. **Phase 1 makes the edge
dialable so workers can upgrade outbound — it works for every NAT'd worker,
including today's fleet. Phase 2 makes the worker itself dialable via a real
router port mapping, restoring the pure pull path.** Phase 1 does not depend on
Phase 2 and ships first.

Throughout, the pull model is untouched: the edge selects the worker from coord's
order limit, initiates every request, and pulls every byte.

---

# Phase 1 — Give the edge server a public libp2p address

## Why

The upgrade machinery is **already implemented in go-libp2p and already
triggering**. Worker logs show `beginDirectConnect` and `attempting direct dial`
on every inbound relayed connection. It fails on one missing input:

```
attempting direct dial ... addrs=[]
attempt to perform DirectConnect failed
  err="failed to negotiate protocol: protocols not supported: [/libp2p/dcutr]"
```

Both failures share one cause. The edge's libp2p host listens on
`/ip4/0.0.0.0/tcp/0` — ephemeral, never announced — so it is stuck forever in
`waitForPublicAddr` (Loki confirms: logs `"waiting until we have at least one
public address"`, never `"Host now has a public address"`). It therefore never
registers the `/libp2p/dcutr` handler (`svc.go:139`) and has no address for
identify to hand the worker.

Workers are **already ready** — they log `"Host now has a public address"`.

With a public address on the edge, the worker's direct-dial short-circuit fires
(`holepuncher.go:119-138`): a plain outbound dial, worker → edge. **No hole
punching**, so this network's symmetric NAT (one local port → public ports 37634
vs 9999, across two alternating public IPs) is irrelevant.

The worker knows which edge to dial from `conn.RemotePeer()` on the connection
the edge itself opened — no registry, no discovery, no config, and a worker never
contacts an edge that is not already talking to it.

## Changes

**1. `go-sdk` — finish the half-built listen option.** `options.go:80-85` defines
`WithP2PListenAddr`, but it is **dead code**: `o.p2pListenAddr` is never read, and
`connector.NewP2PConnectorFromIdentity`
(`internal/grpcutil/connector/p2p_connector.go:60-63`) takes only identity and
resource limits, hardcoding `/ip4/0.0.0.0/tcp/0` at lines 99-101. Add a
listen-address parameter fed into `libp2p.ListenAddrStrings`, and pass
`o.p2pListenAddr` from `client.go:133`. Empty ⇒ `0.0.0.0:0`, existing callers
unaffected.

**2. `go-sdk` — add an announce address.** New `WithP2PAnnounceAddr(addr string)`
applied via `libp2p.AddrsFactory`, appending an operator-supplied public
multiaddr. **Do not use `ForceReachabilityPublic()`** — that makes a host assert
reachability it may not have. No existing `AddrsFactory` use in either repo.

**3. `edgeserver` — expose both as config.** `Server.P2PListenAddr` (e.g.
`0.0.0.0:7788`) and `Server.P2PAnnounceAddr` (e.g. `/ip4/<edge-ip>/tcp/7788`) in
`edgeserver/config.go`, wired in `edgeserver/server.go`. Both empty ⇒ today's
behaviour. `Server.ResourceLimits` already exists (`config.go:111`,
`sdkResourceLimits` at `:184`), so inbound connections are already bounded.

**4. Deployment — the load-bearing half.** Publish the fixed port from the edge
container and open it on the VPS firewall. **Without this the code does nothing,
and it fails silently.**

## Tests

`client_internal_test.go:19` `TestNewClient_CopiesAllOptionsFields` exists
precisely to catch "an option was defined but never reached what uses it" — the
exact bug `WithP2PListenAddr` has. It inspects `Client` fields, so it cannot catch
this one; these options are consumed at connector construction.

- New connector-level test: build a host with a fixed listen addr + announce
  addr, assert `host.Addrs()` contains both. This is the test that would have
  caught the dead option.
- Assert the empty/default path still yields an ephemeral `0.0.0.0:0` listener.

---

# Phase 2 — Make NAT'd workers genuinely dialable via UPnP / NAT-PMP

## Why this and not a config-supplied address

A worker whose operator already knows a public address and can forward a port can
set `HAVE_PUBLIC_ADDRESS=true` today and get plain TCP with **no relay at all**.
So a config-supplied address adds almost nothing — only relay fallback.

The population that matters is workers behind ordinary consumer NAT with no
operator intervention. For them the only mechanism that produces a **genuinely
dialable** address is asking the router for a port mapping.

Critically, **an observed address is not a dialable address.** Identify only
reports the mapping one peer saw for one flow; on a symmetric NAT that mapping is
per-destination, so advertising it manufactures an address that looks valid and
isn't. Phase 2 must not be built on observed addresses.

### What libp2p already gives us

- `libp2p.NATPortMap()` (`options.go:418-422`) — *"will attempt to open a port in
  your network's firewall using UPnP."* **Not enabled anywhere in either repo**
  (`grep NATPortMap` → zero hits).
- The resulting mapping is a **first-class advertised address**:
  `addrs_manager.go:539` calls `natManager.GetMapping(listenAddr)`, and the
  comment at `:450` notes *"AllAddrs may ignore observed addresses in favour of
  NAT mappings"* — mappings are **preferred over** observed addresses.
- `dial_ranker.go:23,50`: *"Dialing relay addresses is delayed by 500 ms, if we
  have any non-relay alternatives."* `host.Connect` takes an `AddrInfo` with many
  addresses and ranks private → public → relay. **We write no direct-first logic,
  no racing, no fallback — we only supply more than one address.** A failed
  mapping costs one bounded 500 ms delay, then the relay succeeds.

So a worker that successfully maps a port ends up advertising a real public
address *and* its relay circuit, and libp2p picks correctly. A worker behind
CGNAT or with UPnP disabled simply gets no mapping and behaves exactly as today.

### No schema or wire change

The advertised address is already a `;`-separated list (`internal/vo/peer_url.go:29`
documents the combined form) carried end-to-end as an opaque string: worker →
coord `CheckInRequest.address`, coord `workers.address TEXT`, coord → downloader
`OrderLimit.worker_address`. **No DB migration, no proto change** — only the
string's content, the parser, and the dialer.

## Changes

**5. Worker — enable NAT port mapping.** Add `libp2p.NATPortMap()` to the worker's
host options (`worker/server/server.go:163-172`), behind a config flag defaulting
on. Note `EnableAutoNATv2()` is already set at `:165` but is neutered by
`ForceReachabilityPrivate()` at `:202` (which becomes
`autonat.WithReachability(private)`, `config.go:752-753`) — leave that alone; the
mapping path does not depend on AutoNAT's verdict.

**6. Worker — advertise the mapped address alongside the circuit.**
`worker/relay/reserve.go:83-103` `selectAdvertisedLocator` deliberately skips
non-circuit addresses; add a sibling that *collects* them. Filter `host.Addrs()`
to public (`manet.IsPublicAddr`), non-circuit, colon-free; cap at 2; emit direct
candidates first, then the circuit locator, joined with `;`, reusing
`externalAddr` (`reserve.go:72-74`) per segment.

**7. Worker — re-derive the address at check-in.** `worker/peer.go:405-418` has a
comment describing exactly this with **no implementing code**, and
`worker/contact/service.go:238-249` `UpdateSelf` never touches `Address` — the
advertised address is frozen at construction (`peer.go:262-270` says so). A UPnP
mapping is established asynchronously after startup, so without re-derivation it
would never be advertised. Add a re-derive callback before each check-in,
mirroring the `SetRelayAddrsCallback` pattern already wired at `peer.go:406`. This
also fixes the pre-existing staleness bug where relay migration is invisible
until restart.

**8. `internal/vo/peer_url.go` — carry multiple p2p addresses.** The parse loop
does `p.P2PAddr = connectionInfo[2]`, so a second `p2p:` segment silently
overwrites the first. Add `P2PAddrs []string` (append each), keep `P2PAddr` as the
first for compatibility, add `P2PAddresses() []string`. Keep the
`strings.Split(c, ":")` length-3 rule, which forbids colons in a multiaddr:
**`/ip4/` and `/dns4/` only**, never `/ip6/` or `/dns6/`.

**9. Dialer — hand libp2p every address.** `connector/dial.go:51` builds one
`AddrInfo` via `peer.AddrInfoFromString`. Build `AddrInfo{ID, Addrs: [...all]}`
instead. Update `p2p_connector.go`'s `DialContext`, which passes the single
`url.P2PAddress()`. Leave `LibP2PDialer.Dial`'s `network.WithAllowLimitedConn` —
required for the relay fallback.

**Do not touch `HybridConnector`** (`hybrid_connector.go:27-42`): a hard
first-match switch where TCP wins with **no** fallback to p2p. We add only `p2p:`
segments, so it is unaffected. Emitting a `tcp:` segment for a NAT'd worker would
make every dial try an unreachable address with no fallback — the trap to avoid.

## Tests

- `internal/vo/peer_url_test.go` — **does not exist today.** Cover: single `p2p:`
  (unchanged), combined `tcp:…;p2p:…`, new multi-`p2p:`, and `/ip6/` proving it is
  rejected rather than mis-parsed.
- Extend `worker/relay/reserve_test.go` — it already asserts the exact
  `p2p:<id>:<multiaddr>` string shape.
- Dialer test asserting every advertised address reaches `AddrInfo.Addrs`.
- `internal/testplanet/run_test.go:36` `TestWorkerCheckIn` — assert the stored
  address carries both a direct and a circuit segment when a mapping exists.
- `internal/testplanet/upload_test.go:78` `TestUploadDownload` must pass
  unchanged, proving relay-only workers are not regressed.

---

# Verification (both phases)

**Step 1 — external reachability. Do this first; nothing downstream works without
it.** `dev/relay-ping/main.go:455-467` already accepts
`/ip4/<host>/tcp/<port>/p2p/<peerID>` meaning *"dial that peer directly, no relay
hop"*:

```
go run ./dev/relay-ping -target /ip4/<edge-public-ip>/tcp/<port>/p2p/<edgePeerID>   # phase 1
go run ./dev/relay-ping -target /ip4/<mapped-ip>/tcp/<mapped-port>/p2p/<workerID>   # phase 2
```

For Phase 2, first confirm a mapping was actually obtained — a worker on CGNAT or
a router with UPnP disabled will get none, which is expected, not a bug.

**Step 2 — the edge admits it has an address.** `log.libp2p-level` is already
wired: `{role="edgeserver1"} |= "Host now has a public address"`

**Step 3 — workers upgrade.** `internal/p2pmonitor/observer.go:128-134` already
detects and logs it: `{app="coord"} |= "relayed connection upgraded to direct"`

**Step 4 — measure.** `./dev/bench/download-test.sh -a both -c 8 -s 30MiB` against
the current baseline on **Grafana → Testing → Download Benchmark**.

**Metrics that decide success:**
- `p2p_direct_upgrades_total` rises above 0 for the first time (phase 1).
- `p2p_direct_peers` per worker rises above 1 — today exactly 1, the relay.
- `relay_bytes_relayed_rate` falls for the same workload.

# Risks

- **Deployment is as load-bearing as the code** (phase 1) and fails silently.
  Step 1 exists to catch exactly that.
- **The edge becomes publicly dialable** (phase 1). Peer identity is verified by
  peer ID and `ResourceLimits` bounds connections/streams, but this is a real
  change in exposure and should be deliberate.
- **UPnP is not universal.** CGNAT, disabled UPnP, and enterprise networks yield
  no mapping — those workers stay relay-only, exactly as today. Expect the current
  Docker fleet to get nothing (containers have no router to ask).
- **UPnP is a security consideration** — the worker asks the router to open a port
  into the host. It should be a documented, disableable config flag.
- **In-flight streams do not migrate.** A running download stays on the relay;
  subsequent fetches go direct. Converges quickly for many short piece fetches.
- **Never advertise observed addresses.** They are per-flow SNAT bindings on a
  symmetric NAT — public-looking and undialable. Only mapped or configured
  addresses may be advertised.
- **Address-format brittleness**: `NewPeerURLFromIDAndAddress` splits on `:` and
  demands exactly 3 parts; `/ip6/` or `/dns6/` breaks parsing. Filter and test.
- **SDK re-pin required**: the edge dials through `go-sdk`. `depin`'s own
  `internal/grpcutil/connector/` and `internal/vo/` are near-identical copies used
  by coord — keep them in sync.
