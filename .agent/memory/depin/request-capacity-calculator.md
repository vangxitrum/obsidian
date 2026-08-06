---
type: reference
tags: [depin, edge, capacity, calculator, html, rps, admission]
created: 2026-08-03
agent: main
---

# Edge request capacity calculator

Standalone interactive page:
`dev/bench/request-capacity-calculator.html` (allowlisted in `.gitignore`). It
uses the repo's dark relay-status visual language and has no runtime dependencies.

The model calculates sustainable incoming RPS as the minimum of request/atomic
stream admission, worker capacity, relay payload, HTTP egress, edge CPU, coordinator
RPS, and proxy concurrency. It distinguishes CDN hits (bypass edge) from edge
cache hits (bypass worker/relay only), accounts for K+tail erasure amplification,
retry/protocol overhead, ranges, segments, service time, headroom, background
windows, and all SDK/libp2p stream ceilings.

The operator-facing admission input is max concurrent downloads, not SDK global
streams. The model derives SDK streams as `max concurrent downloads * (K + tail)`
and still takes the minimum with the three libp2p ceilings.

Core equations:

- `window = K + longTail`
- `wholeWindows = floor((min(SDK, system, connection, peer) - reserved) / window)`
- `missService = fixed + segments*setup + responseMiB/perRequestMiBps`
- `admissionRPS = usableWindows*(1-headroom) / missService / missFraction`
- `ingressMiB = responseMiB*(K + tail*completion)/K*retry*protocol`
- each network ceiling is `capacityMiBps / bytesPerRequest / trafficFraction`
- `overallRPS = min(all ceilings)`

Observed defaults reproduce the 0.45MiB / 3.36s baseline: 14 concurrent downloads
derive 490 SDK streams and produce 14 windows / 3.33 req/s at 20% headroom; 10 req/s
requires 42 downloads / 1,470 streams without cache offload. The page includes Observed, Load collision, and
10 RPS presets, editable basic controls, hidden advanced variables, formula and
demand tables, generated recommendations, localStorage persistence, and JSON
copy. Lavish browser review led to simplifying the initial dense version; user
approved the simplified artifact. `html-validate` and `git diff --check` pass.
All non-admission ceilings also apply the selected operating headroom. The
libp2p per-connection ceiling is modeled separately because the relay topology
can make it the effective minimum even when SDK/system/peer limits are higher.

Related: [[edge-load-test-production-contention]],
[[gosdk-streaming-reconstruction]].
