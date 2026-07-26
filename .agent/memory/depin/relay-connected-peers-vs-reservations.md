---
type: fact
tags: [relay, grafana, observability, circuitv2, autorelay]
created: 2026-07-21
agent: main
---

The relay Grafana "Connected peers" stat panel (monitoring/grafana/dashboards/relay/relay-overview.json,
panel id 13) is bound to `relay_connected_peers{field="ravg"}`, which is
`len(host.Network().Peers())` in coord/relay/observer.go:currentSample — a raw
libp2p connection count on the relay's host. This is NOT the same population as
peers holding a live circuit-v2 HOP reservation on that relay
(`relay_reservations_active`, from the metricsTracer in coord/relay/tracer.go).

A worker can be "connected" to a relay (open TCP/QUIC conn) without a reservation
there: worker/peer.go's `RelayNumRelays` defaults to 1 (one reservation per
worker), but the full relay fleet is handed to
`libp2p.EnableAutoRelayWithStaticRelays` (worker/server/server.go:197) as the
candidate pool. go-libp2p's AutoRelay relayFinder keeps connections open to
backup/candidate relays it probed but didn't end up reserving on. Dialing THROUGH
the relay to one of those "connected but not reserved here" peers gets rejected
at the relay's HOP protocol with `pbv2.Status_NO_RESERVATION` (code 204) — see
coord/relay/tracer.go ConnectionRequestHandled, counted in
`relay_circuit_rejections{status="NO_RESERVATION"}`.

Confirmed live on the local devnet monitoring stack (Prometheus at
127.0.0.1:9090, part of `monitoring-*` docker compose) on 2026-07-21:
`relay_connected_peers` flat ~95 for 3h+ while `relay_reservations_active` was
flat at exactly 44 for the same 3h window, then jumped to ~83-87 in the last
~10min of observation — and cumulative `relay_circuit_rejections{status="NO_RESERVATION"}`
was already at 7020. This matches a user report of "94 connected peers in
Grafana, only 46 respond to a through-relay ping test, rest return 204."

**Why:** the panel's label ("Connected peers") reads as "peers reachable through
this relay," but the query only proves an open connection exists, not that a
circuit reservation exists. `relay_reservations_active` (already emitted, shown
on panel id 12 "Reservations & circuits (active)") is the correct metric for
relay-reachability capacity/health, but it's a secondary line-chart panel easy to
miss next to the prominent "Connected peers" stat.

**How to apply:** when someone asks "why does the relay dashboard/peer-count not
match ping/reachability tests," check this gap first before assuming a bug in
reservation renewal or AutoRelay. If asked to fix the dashboard, prefer relabeling
panel 13 to something like "Connected peers (raw libp2p, includes non-reserved)"
and/or promoting a "reservations_active" stat panel as the primary reachability
number, rather than changing relay/worker behavior. Related: [[relay-domain-locator-egress]],
see also the coord/relay tracer/observer code (coord/relay/tracer.go,
coord/relay/observer.go, coord/relay/rttprobe.go) which is fully wired
(contradicts an older, now-outdated note in [[contract-file-tags-usage]] that
called relay observability "a phantom memory, uncommitted work lost" — it is
present and live as of this date, likely landed since).

**CORRECTION (see UPDATE below):** the "reserved on a different relay in the
fleet" explanation two paragraphs up was a reasonable first guess but turned out
to be WRONG for the actual incident investigated live — this deployment only has
ONE relay instance reporting metrics, and the real cause was a resource-manager
ceiling, not multi-relay reservation spreading. The metric-semantics gap (raw
conns vs reservations) itself is still correct and general; only the "why do
some peers lack a reservation" explanation was revised.

**UPDATE 2026-07-21 — actual live root cause found on this deployment's test relay.**
Live relay logs (docker service `coord-relay`) showed periodic mass
disconnect/reconnect bursts: ~40 distinct worker peer IDs sharing just two source
IPs (`113.176.62.214`, `118.69.133.193` — a test fleet, many worker processes
behind one NAT/host, confirmed by the user) all logged `worker disconnected`
within under a second, then all reconnected + got a **new** (non-renewal)
reservation ~2-3s later. This repeated every few minutes. Ruled out: TTL expiry
(`worker reservation(s) closed count=0` fires almost every tick — nothing is
hitting the 2h TTL) and per-IP reservation/connection caps
(`reservation_rejections=0` throughout; `resource-limits.max-conns-per-ip`
defaults to 512, `MaxReservationsPerIP` defaults to 200 — both far above ~90
total).

Prime suspect: `coord/relay/peer.go:79-87`'s own comment predicts this exact
symptom — go-libp2p's **autoscaled system-wide resource manager**
(`internal/p2putil.ResourceLimitConfig`, `Unlimited=false` and
`MaxConnections`/`MaxInboundStreams`/`MaxOutboundStreams=0` by default) sizes its
connection/stream ceiling off the relay container's RAM/FD ulimit, not the fleet
size; hitting it resets worker connections and drops their reservations,
"observed as reservations_active plateauing below the worker count" — exactly
what happened (44 stuck vs 95 connected). A worker-side mass restart only
refills the budget temporarily (a synchronized burst of reconnects gets
re-admitted), which matches the user's report that restarting every worker on
that IP "fixes" it — until the cycle presumably recurs.

Fix (not yet applied/confirmed — user needs to check the deployed relay's actual
flags/container memory limit first): set `--resource-limits.unlimited=true`, or
explicitly raise `--resource-limits.max-connections` /
`--resource-limits.max-inbound-streams` / `--resource-limits.max-outbound-streams`
above -1/expected peak, on the `coord relay` process. Flag names confirmed via
`go run ./cmd/coord relay --help` in this repo (cobra + internal/cfgstruct,
prefix `resource-limits.`, hyphenated field names).

Also found in the same log dump, unrelated but real: the relay's telemetry
remote_write push to `https://prometheus.tunnel.appdemo.cyou/api/v1/write` was
intermittently failing with `tls: internal error` (`pkg/telemetry/client.go`) —
this alone explains Grafana showing a stale/lower reservation count than a live
`/api/v1/query` against the same Prometheus (compounded by panel 12 using
`field="ravg"`, monkit's smoothed value, vs `field="recent"`, the literal last
sample).
