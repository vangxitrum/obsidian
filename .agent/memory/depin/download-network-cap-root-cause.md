---
type: fact
tags: [depin, download, throughput, network, relay, edge, workers]
created: 2026-08-03
agent: main
---

# Download throughput ceiling root cause

The apparent ~8-9MiB/s direct-SDK ceiling is not a configured byte-rate limit.
It is the useful-payload ceiling of the shared public worker-to-relay path; the
edge's ~4MiB/s is then explained by full-segment store-and-forward across two
serial network legs.

## Runtime topology

- Coord API, relay, and edge are on `68.183.189.51`; edge reaches API by Compose
  DNS and the relay hostname resolves locally on that host.
- Workers are three 50-worker fleets (`worker-*`, `oldwin-worker-*`, and
  `transcribe-worker-*`), confirmed by 150 active relay reservations.
- On the accessible `worker-*` host, all 50 containers are active on one Docker
  bridge. Their runtime relay cache contains `/dns/relay/...`, but container DNS
  maps `relay` to public `68.183.189.51`.
- Host route to the relay is through default gateway `10.0.0.1`; TCP MTR traverses
  VNPT/public transit to Singapore at roughly 29-47ms. This is not a VPS-provider
  private-network route. The workers' shared local network is not used for piece
  payloads because workers never exchange pieces with one another.
- Edge reports about 128 relayed-only peers and one direct peer; direct upgrades
  are not materially carrying downloads.

## Measurements

- Same 8x80MiB corpus, direct SDK: 8/8, 68s, **9.41MiB/s aggregate**.
- Same corpus through edge: 8/8, 142s, **4.51MiB/s aggregate**.
- One 80MiB direct SDK baseline: about **8.0MiB/s**.
- One 80MiB edge request: 21s, **3.81MiB/s**.
- The relation is harmonic: two serial ~8MiB/s legs yield
  `1 / (1/8 + 1/8) = 4MiB/s`, matching both single and 8-way observations.
- During the edge run the accessible worker host first transmitted piece data
  toward the public relay (bursts up to ~10.6MB/s), then received HTTP bodies
  from edge (bursts up to ~14.9MB/s). During direct SDK, RX and TX overlapped.
- Host NIC is 1Gbps full-duplex and stayed below ~20% utilization. The physical
  NIC is not the cap; any egress shaping/congestion is beyond it (gateway/ISP/
  public transit/relay path).
- Successful worker bytes during the edge run were balanced: oldwin 298.1MB,
  transcribe 253.7MB, worker 230.2MB. Placement is not concentrated on one fleet.
- Relay payload can burst above 13MB/s. No SDK, worker, relay, or edge byte-rate
  setting at 8MiB/s exists.
- Edge bounded gRPC reuse default 2 produced cache hits but did not change the
  4.51MiB/s result, ruling connection setup out as the hard ceiling.

## Why edge halves throughput

`piecedownload.Manager.Fetch` buffers every required piece completely before
`segmentdownload` can decode/decrypt and write plaintext. Segment processing is
strictly serial, so edge mostly performs:

1. worker fleet -> public relay -> edge (fetch complete segment)
2. edge -> HTTP client (serve buffered/decrypted segment)
3. repeat for the next segment

The network legs are added in wall time rather than pipelined. Segment prefetch
was removed on 2026-08-03 because it increased full-piece buffering and fan-out
without removing this ~2x store-and-forward penalty. That requires stripe-level
streaming decode/output or an edge/CDN cache hit.

## Practical fixes

1. Stripe-level streaming piece reconstruction was implemented in the Go SDK on
   2026-08-03 for Reed-Solomon segments. Live 8x80MiB edge throughput improved
   from 4.51MiB/s to 5.05-5.29MiB/s, but remains below the new-SDK direct result
   of 8.10MiB/s; see [[gosdk-streaming-reconstruction]].
2. Keep/enable edge or CDN object caching for repeat downloads, bypassing the
   worker/relay leg entirely.
3. To raise the direct SDK ceiling, improve the workers' shared WAN uplink/public
   route or distribute fleets across genuinely independent uplinks/regions and
   relays. A private network among worker hosts does not help this protocol.
4. If private addressing is desired, add routed peering/VPN/direct reachability
   between edge and worker networks plus requester-aware address fallback. The
   current coordinator stores one opaque worker address and cannot select a
   private direct address with relay fallback.

Related: [[download-benchmark-harness]], [[grpc-pool-reuse-relay-regression]],
[[relay-domain-locator-egress]], [[gosdk-streaming-reconstruction]].
