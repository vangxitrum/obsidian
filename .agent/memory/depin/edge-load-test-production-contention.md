---
type: fact
tags: [depin, edge, load-test, contention, admission, metrics, relay]
created: 2026-08-03
agent: main
---

# Edge load test slows production through admission contention

Controlled window `1785749784..1785750033` / benchmark
`dl-metrics-contention-20260803` ran 8x80MiB against the live 512-cap edge. All
eight passed, but needed 207-234s and only 2.74MiB/s aggregate because the same
edge was simultaneously serving heavy normal small-object traffic.

The primary cause of the unrelated-service slowdown is the SDK's shared atomic
segment admission budget, not CPU:

- Every RS segment reserves K+6 = 35 global piece permits.
- The benchmark occupies `8 * 35 = 280` of the edge's 512 permits.
- Only `232 / 35 = 6` complete windows remain (22 permits cannot form a window).
- Before load, normal small-object traffic completed 706 requests / 249s =
  2.84 req/s with 3.36s average latency, implying about 9.5 concurrent requests
  by Little's Law. It therefore normally needs about ten windows, but gets six
  during the benchmark.
- Normal 300-534KiB requests fell from 706 to 473 completions, average latency
  rose 3.36s -> 16.81s (5x), and reported throughput fell 0.301 -> 0.053MiB/s
  (82%).
- Phase decomposition proves queueing: unmeasured/pre-piece time rose 1.88s ->
  14.28s and explains about 12.4s of the 13.45s latency increase. Actual piece
  time rose only 1.12s -> 2.13s.

Secondary contention exists on the shared relay/network path: relay payload rate
rose from 1.71MB/s average / 2.12MB/s max before load to 4.73MB/s average /
7.08MB/s max during load, and active piece time nearly doubled. But this was not
CPU saturation: edge averaged 0.48 CPU cores, relay 0.14, all 150 workers combined
0.74; relay had zero new circuit rejections. No node-exporter NIC series are
currently ingested, so physical NIC utilization was not measurable from this
Prometheus window.

**How to apply:** Do not run concurrency-8 benchmarks against the production edge
without traffic isolation. Immediate safe load is concurrency 4: four benchmark
windows plus the normal ten-window working set fits the 512 budget. Better
production designs are a separate benchmark/canary edge SDK client/process,
reserved/priority admission for normal small-object traffic, and CDN/edge caching.
Simply raising the cap beyond roughly 630 would also require raising libp2p's 512
ceilings and would increase relay/network contention rather than provide isolation.

Related: [[gosdk-streaming-reconstruction]], [[download-network-cap-root-cause]],
[[download-benchmark-harness]].
