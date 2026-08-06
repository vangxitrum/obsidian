---
type: reference
tags: [depin, edge, relay, capacity, vps, scaling]
created: 2026-08-03
agent: main
---

# VPS capacity sizing for edge and relay

Hardware dimensions alone do not determine request capacity. For the current
uncached 0.45MiB HLS workload, measured successful service is 4.64-4.89s at 21
slots, yielding 4.30-4.52 req/s hard or 3.44-3.62 req/s with 20% headroom. Plan a
combined 1-vCPU/4GB relay+edge VPS for about 3 sustained req/s until that exact
host/provider is benchmarked; treat 3.5 req/s as the measured planning ceiling and
4.4 req/s as saturation, not SLA capacity.

The current RS path relays approximately `responseMiB * 35/29 * 1.05`, or about
0.57MiB per 0.45MiB request. At 3 req/s, combined relay ingress + edge HTTP egress
is about 24.5Mbps payload and 7.57TiB per 30-day month. At 3.5 req/s it is 28.6Mbps
and 8.83TiB/month. At 10 req/s it is 81.6Mbps and 25.22TiB/month. Provider transfer
quota and sustained route quality matter more than advertised port speed.

Evidence: an earlier mixed production+bulk window used only about 0.48 edge cores
and 0.14 relay cores, but host CPU/RSS/NIC metrics are not currently exported, so
4GB RAM and a 1-vCPU combined-host ceiling are not directly verified. The relay
path sustained about 3.99MB/s average / 4.84MB/s peak in the latest contention run
and 7.08MB/s peak historically, while latency worsened materially as load rose.

**Scaling guidance:** For the current 0.45MiB workload, use 21 slots on a combined
1-vCPU/4GB host and alert before 3.5 sustained req/s. Scale when real weighted
failure exceeds 1%, admission rejects persist, p95 exceeds 1.5x baseline, relay
payload exceeds 70% of a provider-specific sustained benchmark, or host CPU/RSS
exceeds 70%. Scale edge and relay separately: adding edge replicas does not fix one
shared relay path. A 70% CDN/edge-cache hit rate reduces 10 incoming req/s to 3
origin misses/s, which fits the measured 1-vCPU origin envelope much more credibly
than raising uncached concurrency to roughly 60 slots.

Related: [[edge-request-admission]], [[request-capacity-calculator]],
[[edge-load-test-production-contention]].
