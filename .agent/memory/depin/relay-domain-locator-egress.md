---
type: decision
tags: [depin, relay, networking, egress, libp2p, dns, cost]
created: 2026-07-21
agent: main
---

# Relay /dns4 locator + split-horizon DNS to cut relay→edge egress

**Goal:** stop paying billed public egress on the `relay → edge` hop. The relay
hairpins piece data; when our own in-VPC edge/gateway pulls pieces from a NAT'd worker,
`relay → edge` leaves the relay's public IP and is billed. Edge, coord, relay share one
VPC (confirmed), so that leg can be free if it rides the private network.

**Chosen design (Approach A):** put a `/dns4/<host>` name (not a raw IP) in the worker's
relay circuit locator, and serve it via **authoritative split-horizon DNS** — internal
view (VPC source ranges) → relay **private** IP (free), external view → relay **public**
IP (external go-sdk clients unchanged). A domain alone saves nothing; the split-horizon
view is what moves the bytes. Dual-address link (Approach C: private+public addrs in one
link, libp2p picks reachable) was **weighed and deferred** — it adds link-format/parser
changes and an external-client dial-timeout on the blackholed private addr.

**Why it works / key code facts:**
- The locator is built in `worker/relay/reserve.go` from the operator-configured relay
  addr (coord `RelayAddrs`, `coord/contact/service.go:49`, distributed via
  `GetRelayAddrs` + check-in echo; dev override `worker.relay-addrs` in
  `dev/workers/p2p/worker-*/config.yaml`).
- That locator string is stored by coord and signed into the download **order limit**
  (`WorkerAddress`, `coord/order/signer.go:216`) — so it IS what the edge/SDK dials.
- libp2p resolves `/dns4/` at dial time natively; no dial-path change. Peer identity is
  the peer ID (Noise/TLS), so the host part is a pure transport locator — no cert/SAN
  impact.
- **The one real code change:** `WaitForRelayReservation` currently reports whatever
  AutoRelay *advertises* (`advertisedCircuitAddr`), which for a public relay may be the
  resolved IP, not your `/dns4/`. Fix = new `selectAdvertisedLocator(hostID, hostAddrs,
  relays)`: use the advertisement as the "reservation live" signal but rewrite the relay
  component to the *configured* relay addr, matched by relay peer ID
  (`relayIDFromCircuit` via `Decapsulate("/p2p-circuit")` + `AddrInfoFromP2pAddr`;
  `findConfiguredRelay`). Domain then survives verbatim into the order limit.
- **Gotcha:** use `/dns4/` ONLY. `internal/vo/peer_url.go` splits on `:`; `/dns6/` or
  `/ip6/` would break it. `/dns4/` is colon-free — no parser change needed.

**Status (2026-07-21):** brainstormed → spec → plan, all in the vault. NOT yet
implemented. Spec: `Projects/depin/specs/2026-07-21-relay-domain-locator-split-horizon-design.md`.
Plan: `Projects/depin/plans/2026-07-21-relay-domain-locator-plan.md` (4 tasks: reserve.go
TDD change; config→/dns4; authoritative split-horizon DNS + relay 0.0.0.0 listen + public-
egress monitor; e2e). Awaiting user's execution choice (subagent-driven vs inline).

**Op note:** the hermes router hit its **monthly** request cap on
`kr/claude-sonnet-4.5-agentic` (HTTP 402 `MONTHLY_REQUEST_COUNT`) on 2026-07-21, so the
mandated "route vault writes through hermes" channel was down; fell back to direct file
writes (disclosed to user). Related: [[worker-portable-package]], relay hairpin throughput
work (see depin auto-memory `project_throughput_diagnosis`).
