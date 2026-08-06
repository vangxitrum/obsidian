---
type: decision
tags: [depin, edge, admission, concurrency, libp2p]
created: 2026-08-03
agent: main
---

# Edge capacity is configured as concurrent downloads

The edge exposes `server.max-concurrent-downloads` (default 14) instead of a
user-facing global piece-stream limit. Only requests fetching a body consume a
slot; conditional 304 and HEAD requests bypass the gate. An excess request is
rejected before entering go-sdk with HTTP 503 and `Retry-After: 1`, and increments
`edge_download_rejected`.

The edge retains the SDK global limiter as an internal safety boundary. Its piece
budget is derived as `max concurrent downloads * 35`, where 35 is the current
29 required + 6 long-tail RS window. Default 14 therefore yields 490 piece
streams, the same 14 complete windows as the former 512 setting. Default libp2p
system/connection/relay-peer outbound ceilings derive as the piece budget + 22
protocol/coordinator headroom, preserving the prior default of 512 and scaling
with request admission.

**Why:** Operators reason in requests, while the raw piece count leaked the SDK's
implementation and caused excess requests to queue invisibly inside piece
admission. Immediate request-level backpressure follows Storj's live-request
rejection pattern.

**How to apply:** Tune `--server.max-concurrent-downloads`; do not configure raw
edge SDK streams. Keep explicit libp2p limits at or above the derived budget when
overriding their defaults. Request admission controls concurrency, not sustainable
RPS or relay bandwidth, and a shared gate still does not isolate benchmark traffic.

Related: [[edge-load-test-production-contention]],
[[request-capacity-calculator]], [[gosdk-streaming-reconstruction]].

## First live benchmark snapshot

At 2026-08-03 17:42:20 +0700, the still-running distributed k6 test
`hls-run-001` had 50 VUs (25 on each of two generators). The new 14-slot edge
returned 953 HTTP 200 responses and 513 intentional HTTP 503 responses: 35.0%
rejected, with zero recorded edge download errors. Successful request duration
was 3.40-3.44s average and 6.01-6.08s p95; aggregate p95 TTFB was about 4.49s.
Rejected requests returned in about 49ms average / 168-176ms p95. Average served
object size was about 458KiB. This proves overload now fails fast instead of
queueing inside SDK admission, but 14 slots are insufficient for this offered
load. At roughly 5 req/s and 3.42s successful service, the run needs about 18
slots minimum or 21 with 20% operating headroom; increasing slots also increases
relay load and needs a follow-up test.

## 21-slot contention benchmark

On 2026-08-03, the edge was restarted with 21 slots while the distributed
`hls-run-001` workload ran at 50 VUs. A controlled concurrency-4 benchmark then
downloaded four exact 80MiB files during `18:04:31..18:07:09 +0700`
(`dl-slots21-contention-20260803`). All four returned exact HTTP 200 bodies in
141.2-158.5s, 2.03MiB/s aggregate.

Synchronized HLS comparison:

- Baseline `18:03:15..18:04:31`: 250 accepted / 83 rejected, 3.29 accepted
  req/s, 4.89s successful mean, 24.9% rejected, about 1.39MiB/s served.
- During bulk `18:04:31..18:07:09`: 364 accepted / 339 rejected, 2.30 accepted
  req/s, 6.51s successful mean, 48.2% rejected, 1.04MiB/s served.
- Post `18:07:09..18:09:47`: 547 accepted / 187 rejected, 3.46 accepted
  req/s, 4.64s successful mean, 25.5% rejected, 1.54MiB/s served.

Thus bulk contention cut HLS accepted throughput by 30% and served bandwidth by
25%, raised successful latency 33%, and nearly doubled rejection. Relay payload
rose from 1.84MB/s baseline to 3.99MB/s average / 4.84MB/s max during bulk; there
were no edge transfer errors or new relay circuit rejections. The bottleneck is
shared relay/network capacity plus occupied request slots, not failed downloads.

Capacity from Little's Law:

- HLS-only hard ceiling: `21 / 4.64..4.89 = 4.30..4.52 req/s`; with 20%
  headroom, plan for only 3.44..3.62 req/s.
- With four long-lived bulk slots: 17 HLS slots and 6.51s service yield 2.61
  req/s hard / 2.09 req/s with headroom; observed was 2.30 req/s.
- Current offered HLS load was 4.38-4.65 req/s. Without bulk it needs at least
  22 slots average or about 27 with 20% headroom. Under the measured bulk
  contention it needs 29 HLS slots average / 37 with headroom, plus four bulk
  slots: 33 minimum / 41 planned total.
- Ten uncached HLS req/s would calculate to about 59-61 slots with no bulk, or
  82 HLS + 4 bulk slots under measured contention. These are arithmetic demand,
  not achievable proof: more slots increase relay contention and may further
  lengthen service time, so caching or traffic isolation remains required.

## Apparent 100% HLS failure

The 2026-08-03 HLS dashboard's apparent 100% request failure was partly a metric
aggregation trap. k6 exports `k6_http_req_failed_rate` separately by status: every
HTTP 200 series has value 0 and every HTTP 503 series has value 1. Therefore
`max(k6_http_req_failed_rate{...})` always displays 100% as soon as any rejected
series exists; it is not the aggregate request failure ratio. Calculate the real
recent ratio from counters:

`sum(increase(k6_http_reqs_total{expected_response="false"}[$window])) /
sum(increase(k6_http_reqs_total[$window]))`.

There was also one genuine near-100% 15-second interval around 18:15:15 +0700:
the edge was deliberately restarted then (`edge server listening` at 18:15:15.529),
and that scrape interval had 1 HTTP 200, 11 HTTP 502, and 10 HTTP 503 responses.
No new 502s occurred afterward. Six edge errors were broken pipes from clients or
the proxy closing in-flight responses, not worker/RS failures. Subsequent live
30-second samples were 79-86 HTTP 200 versus 2-6 HTTP 503 (about 2-7% actual
failure), with all 50 VUs active.
