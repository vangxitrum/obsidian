---
type: spec
tags: [depin, relay, networking, egress, libp2p, split-horizon-dns, cost]
created: 2026-07-21
status: design locked — pending implementation plan
project: depin
decisions:
  approach: "A — domain + split-horizon DNS (domain-only for v1); C documented as follow-on"
  dns: "authoritative split-horizon (internal vs external view by source)"
---

# Relay domain locator + split-horizon DNS to move edge↔relay onto the VPC

## Problem

Worker download links are circuit-relay locators of the form:

```
p2p:<worker-id>:/ip4/68.183.189.51/tcp/7781/p2p/<relay-id>/p2p-circuit
```

The relay hairpins the data plane. When our own edge/gateway pulls pieces from a
behind-NAT worker, the bytes flow `worker → relay → edge`, and the `relay → edge`
leg leaves the relay over its **public** IP — billed egress on our cloud account.
Because the relay's public IP is baked into the link, every consumer (including our
in-VPC edge) dials the relay publicly, so we pay for traffic that never needed to
leave our private network.

## Goal

Move the `relay → edge` leg (and the analogous `coord → relay` audit/repair/GC leg)
onto the free VPC network, **without** breaking external go-sdk clients that dial the
exact same worker links from the public internet.

## Confirmed constraints

- Edge/gateway, coordinator, and relay are (or will be) in **one cloud VPC** where
  internal transfer is free.
- Worker links are consumed by **both** our in-VPC edge **and** external SDK clients
  (go-sdk direct downloads via client-minted tickets). One link must therefore serve
  two audiences with different routes to the relay.
- Workers are behind NAT / third-party; the `relay ↔ worker` hop is inherently public.

## Background: how the link is built and consumed (code map)

- The relay component of the link comes verbatim from `worker.relay-addrs`
  (coord `RelayAddrs`, `coord/contact/service.go`), delivered to workers via
  `GetRelayAddrs` + the check-in echo (`coord/contact/endpoint.go`,
  `worker/relay/bootstrap.go`).
- The worker composes its external contact address in `worker/relay/reserve.go`
  (`externalAddr` at :72), reported as `peer.ExternalAddr` (`worker/peer.go:281`)
  and stored by the coordinator.
- On download, the coordinator signs the worker address into the **order limit**
  (`WorkerAddress`, `coord/order/signer.go:216`), handed to whoever fetches the piece.
  So the string the downloader dials **is** this link.
- Dial path: `internal/grpcutil/connector/p2p/conn.go` → `host.NewStream` (dials by
  peer ID against addresses in the peerstore). libp2p resolves `/dns4/` multiaddrs at
  dial time natively — no dial-path change needed for a domain.
- Peer identity is the libp2p **peer ID** (Noise/TLS over that ID). The host portion
  of the multiaddr is a pure transport locator with **no cert/SAN meaning**, so it can
  be a domain freely — no TLS/certificate changes.

## Design (Approach A — recommended): domain locator + split-horizon DNS

Replace the relay's raw IP in the link with a domain, and let each dialer's resolver
decide which IP it gets:

| Dialer | Location | Resolves `relay.aioz.io` → | Path | Billed to us? |
|---|---|---|---|---|
| Edge / gateway | in-VPC | relay **private** IP | edge↔relay over VPC | **No** (was yes) |
| Coordinator (audit/repair/GC) | in-VPC | relay **private** IP | coord↔relay over VPC | **No** (bonus) |
| External SDK client | internet | relay **public** IP | over internet | Yes (unchanged) |
| NAT'd worker (reservation) | internet | relay **public** IP | over internet | worker's bill (unchanged) |

Download the edge serves: `worker → relay` (relay ingress, free) then `relay → edge`
(now private, free). Remaining unavoidable public egress: relay↔external-clients and
relay↔NAT-workers.

### Changes required

1. **Config (no code).** Set `worker.relay-addrs` / coord `RelayAddrs` to
   `/dns4/relay.aioz.io/tcp/7781/p2p/<relay-id>`.

2. **DNS — authoritative split-horizon (chosen).** Serve the relay domain from two
   authoritative views selected by source: an **internal view** (VPC source ranges) →
   relay **private** IP, and an **external view** → relay **public** IP. This gives
   central control with no per-host `/etc/hosts` edits and scales as in-VPC consumers
   grow. Hard rule: the private A record must never appear in the external view.
   Nothing in the worker/coord code or config depends on the DNS mechanism — it is
   purely operational.

3. **Code — `worker/relay/reserve.go` (the one real code change).** For a public
   relay the worker currently reports whatever AutoRelay *advertises*
   (`advertisedCircuitAddr`, :122), and go-libp2p may substitute the resolved IP for
   your `/dns4/` there (version-dependent — do not rely on it). Fix: keep AutoRelay's
   advertisement as the **readiness signal** (it proves a live reservation), but
   **rewrite the reported locator's relay component to the configured relay addr**,
   matched by the relay peer ID embedded in the advertised circuit. This preserves the
   `/dns4/` domain verbatim regardless of what IP AutoRelay observed, while keeping the
   guarantee that we only report once a reservation is actually live. The pieces exist
   already: the configured relays (`staticRelays`) and the connectedness check
   (`reserve.go:127`).

4. **`internal/vo/peer_url.go` — no change.** The `:`-split parser handles `/dns4/`
   (colon-free). Flag for awareness: `/dns6/` or raw IPv6 in a locator *would* break
   it; out of scope here.

5. **Relay — verify listen.** Relay must listen on `0.0.0.0:7781` so the private
   interface accepts connections (almost certainly already true).

## Alternative (Approach C): dual-address link — weighed, not recommended for v1

Emit **both** a private and a public relay addr (same relay peer ID) in the worker
link, and let libp2p dial both and use whichever connects — in-VPC picks private,
external picks public. No DNS trickery.

**Pros:** removes the DNS dependency entirely; self-contained in the link; aligns with
the "best-of-both dual-address" direction already noted for throughput work.

**Cons / cost:**
- Link-format change: the link and `peer_url.go` (`PeerURL` currently carries a single
  `P2PAddr`) must carry and parse multiple relay addrs — more surface than Approach A.
- **External clients pay a wasted dial** on the private (RFC1918) addr: from the public
  internet that address is typically blackholed → dial timeout added to every external
  download's connection setup (libp2p races addresses but still spends the attempt).
- Harder to reason about and monitor than a single locator + DNS view.

**Decision (locked at spec review, 2026-07-21):** ship Approach A for v1 — config +
one small deterministic-link code change + the authoritative split-horizon DNS view,
with **no** external-client latency penalty. Approach C is kept as a documented
follow-on, to revisit only if operating the DNS views proves painful or if we later
want the link to be fully self-describing without DNS.

## Savings analysis

- **Eliminated:** `relay → edge` (the flagged leg) and `coord → relay`
  audit/repair/GC — both become free VPC transfer. For download-heavy workloads served
  by our own edge, this removes the dominant relay egress line item.
- **Unchanged (inherent):** relay↔external-SDK-clients and relay↔NAT-workers stay
  public — the cost of serving the open internet and third-party storage.

## Verification plan (e2e, real end-user path)

1. Bring up a worker with the `/dns4/` relay-addr; confirm the address the coordinator
   stores and echoes contains `/dns4/relay.aioz.io/...` (proves change #3 preserves the
   domain end-to-end into the signed order limit).
2. From the edge box: resolve the domain → **private** IP; run a real download through
   the edge; confirm the relay's **public** interface byte counter (or relay egress
   metric) does **not** climb for that transfer.
3. From an external client: resolve → **public** IP; confirm the same download still
   succeeds.
4. From the coordinator host: confirm audit/repair dials the relay over the private IP.

## Risks

- **Misconfigured internal view** (edge resolves to public IP): no correctness break —
  we just silently keep paying. Mitigation: monitor/alert on the relay's public egress
  so a regression is visible.
- **Leaked internal view** (external client resolves to private IP): download fails.
  Mitigation: keep the DNS views strict; the private A record must never appear in the
  public zone.
- **go-libp2p version drift** in AutoRelay's advertised address form: fully mitigated
  by change #3 (we stop depending on it).
- **DNS resolution/caching on dial:** negligible; libp2p caches resolved multiaddrs.

## Decisions (locked at spec review, 2026-07-21)

1. **Approach:** A — domain + split-horizon DNS, domain-only for v1. C (dual-address)
   documented as a follow-on only.
2. **DNS mechanism:** authoritative split-horizon (internal vs external view selected
   by source), not per-host overrides.

## Open (needed before/at implementation)

3. **Domain choice / zone ownership** — exact relay hostname and which zone/authority
   owns it (drives the split-horizon view setup). Placeholder in this spec:
   `relay.aioz.io`.
