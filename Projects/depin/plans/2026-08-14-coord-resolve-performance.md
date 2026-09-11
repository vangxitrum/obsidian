# Coordinator download-resolve performance

## Context

Downloads were slow (~14.9s each) and the edge was returning 503s. A long tracing session
walked the whole stack and every measurement converged on one place.

What was ruled out, with evidence:

- **Not the edge's concurrency limiter.** Raising `MaxConcurrentDownloads` 14 -> 40 took
  effect (inferred in-flight concurrency reached 38.3) and did not help. Capacity is just
  `slots / service time` = `40 / 23.8s` = 1.68 req/s against ~2.08 req/s offered.
- **Not the workers or the network.** Fleet-wide a worker serves a piece in 8ms
  (12,097 bytes avg, ttfb-sent 1ms, 5.4 MB/s). The SDK data path is 0.41s of a 14.9s
  download - 2.7%.
- **Not the database.** `pg_stat_activity` showed one coord backend, `idle in
  transaction`, `wait_event=ClientRead` - Postgres waiting on the coordinator. The whole
  DB is a few thousand rows, so table bloat cannot explain a 1.29s query on a 384-tuple
  table.
- **Not table bloat, not connection-pool starvation** - a single-row upsert in the same
  process at the same instant took 0.1ms while a single-row read took 0.74s.

**It is the coordinator resolve**, `GetDownloadInfoByTicket`, at ~97% of download time,
and underneath it a CPU-starved box: pure-CPU ECDSA signing runs 100-1000x slower than
native (25-31ms per signature vs ~30us), with ~595 involuntary context switches/sec on a
process using 0.19 cores, every coord process pinned to `Processor=1`, and the whole coord
stack drawing ~0.6-0.8 cores.

### What the resolve does

```
verify ticket signature                       1 ECDSA verify
per-file rate check                           free
LoadDownloadData                              2 DB reads (file row + segment list)
per-contract rate check                       free
for EACH segment:
    resolve piece aliases -> workers          overlay cache lookup
    cap candidates (DownloadNodes)            free
    CreateGetOrderLimits                      1 ECDSA SIGN PER PIECE   <- the wall
record bandwidth                              1 batched upsert
```

An order limit is a signed capability: it authorizes one worker to serve one specific
piece, up to N bytes, before an expiry, to whoever holds the matching key. The worker
verifies the coordinator's signature before serving, and each limit carries a per-request
serial that workers reject on replay - which is why limits can never be cached or shared
across requests. The cost is therefore structural: **pieces x 1 signature**.

| step | share | status |
|---|---|---|
| `CreateGetOrderLimits` (signing) | 29% | halved by the `DownloadNodes` cap (80 -> 38 sigs), written, undeployed |
| `LoadDownloadData` (+ segment list) | 19% | removed on repeat/concurrent reads by the lrucache memoization, written, undeployed |
| `AuthorizeFileOwner` | ~1.2s ravg | **redundant** - re-reads the same file row, uncached |
| `contractRepository.GetByID` | ~0.7s ravg | cacheable |
| accounting write | 0.007% | already batched |

The signing path itself was inspected for waste and has none: one protobuf marshal
(`pkg/signing/encode.go:13`) plus one ECDSA P-256 sign (`pkg/signing/sign.go:32`), a
faithful Storj port. (`EncodeOrderLimit`'s larger cumulative total just reflects that
settlement verification calls it too.) The work is real; the CPU makes it expensive.

### Decisions taken

- **More coordinator CPU is available** - so parallel signing is worth building.
- **Single-segment minting is in scope** - a proto + go-sdk change.
- Related: [[2026-08-04-direct-edge-worker-connections]] already moved edge<->worker
  traffic off the relay; that fixed the data path, which is why only the resolve is left.

## Scope

1. Deploy the two already-written, untested-in-prod changes (prerequisite, no new code).
2. Parallelize order-limit signing across cores.
3. Remove the redundant/uncached reads inside the resolve.
4. Mint order limits for one segment per request, Storj-style, with the client fetching
   later segments lazily.

## 1. Deploy what is written (do this first, measure, then continue)

Uncommitted and never exercised in production:

- `coord/file/service.go` - `LoadDownloadData` memoized on `internal/lrucache` with
  request collapsing (an HLS audience requests the same segment at once; collapsing turns
  N identical DB reads into one).
- `coord/order/downloadnodes.go` - `DownloadNodes` tail-tolerance cap ported from Storj
  (`t = k + (n-o)k/o`), 80 -> 38 signatures on the seeded RS(29,52,60,80) placement.

Everything below is measured against that baseline, not against today's production. Ship
and re-measure before building more.

## 2. Parallel signing - `coord/order/service.go`

`CreateGetOrderLimits` (`service.go:158-169`) signs sequentially. `PrivateKey` is a
stateless struct over an `ecdsa.PrivateKey` (`pkg/signing/peers.go:23`) and Go's
`ecdsa.Sign` is safe for concurrent use, so the loop parallelizes with no locking.

Write into an indexed slice, not by appending - limit order must stay aligned with
`pieces[i]`, because the client pairs each limit with its piece number:

```go
limits := make([]*sharedpb.OrderLimit, len(pieces))
g, gctx := errgroup.WithContext(ctx)
g.SetLimit(signingConcurrency)          // GOMAXPROCS, config-overridable
for i, p := range pieces {
    g.Go(func() error {
        l, err := signer.Sign(gctx, p.WorkerID, p.WorkerAddress, p.PieceNum)
        limits[i] = l
        return err
    })
}
```

Deliberate divergence from Storj, which signs sequentially - justify it in a comment:
Storj's satellites scale horizontally so per-request latency is not their constraint;
this coordinator is a single CPU-bound process where it is.

**This does nothing on one core.** It converts added cores into resolve latency, roughly
38 x 28ms sequential divided by usable cores.

## 3. Remove redundant reads

`AuthorizeFileOwner` (`coord/file/service.go:184`) calls `s.store.GetByID(fileUUID)`
directly, then `buildManifest` calls `LoadDownloadData` which reads the same row again -
so the file row is read twice per resolve on the client-ticket path, and the first read
bypasses the new cache entirely.

- Route `AuthorizeFileOwner`'s file read through the same cached loader.
- Add a short-TTL cache for `contracts.GetByID` (owner changes are rare; reuse
  `internal/lrucache`, same shape as `downloadData`). Keep the TTL short so ownership and
  soft-delete changes take effect quickly.

## 4. Single-segment order-limit minting

Today `buildManifest` loops every covering segment and mints limits for all of them.
Storj mints for `segments.Segments[0]` only and returns a single-element
`[]*pb.SegmentDownloadResponse` (`satellite/metainfo/endpoint_object.go:1167`); the
uplink pages the remaining segments' **metadata** and calls `DownloadSegment` lazily as
the reader advances.

**Proto** (`pkg/pb/coord/file/v1/file.proto`):

- New `rpc GetSegmentDownloadInfo(GetSegmentDownloadInfoRequest) returns
  (GetSegmentDownloadInfoResponse)`. The request carries the **same raw download ticket**
  plus the segment number, so authorization reuses the existing verified path rather than
  inventing a second one. The response carries one `DownloadSegment`.
- Add `bool lazy_segments` to the download-info requests. **This is the compatibility
  hinge**: only a client that sets it gets a manifest with limits on the first covering
  segment alone. Every deployed SDK and edge keeps today's full manifest, so nothing
  breaks on deploy.

**Coordinator** (`coord/file/endpoint.go`): when `lazy_segments` is set, `buildManifest`
still emits every covering segment's metadata (sizes, encryption parameters, plain
offsets - the client needs these to place and trim) but calls `buildDownloadSegment` for
the first covering segment only. `GetSegmentDownloadInfo` mints one segment on demand,
reusing `buildDownloadSegment` unchanged.

**Accounting** (`recordDownloadBandwidth`): today it bills the whole manifest's egress and
worker allocation up front. Under lazy minting it must bill per segment as that segment is
minted. This is a genuine behaviour change and arguably a correction - a client that
abandons a download after one segment currently gets billed for the whole object - but it
must be called out and covered by a test, because it moves numbers that feed invoicing.

**go-sdk** (`download.go`): `streamSegments` opens jobs one at a time. A segment whose
limits are absent fetches them first via the new RPC. Add a one-segment **prefetch**:
while segment N streams, fetch segment N+1's limits. Without it, lazy minting trades
coordinator load for one extra round trip of added latency per segment - a bad trade on a
multi-segment file.

**Expected effect, stated plainly:** the HLS objects driving this work are ~465KB against
a 64MB `DefaultSegmentSize`, so they are single-segment and this changes nothing for them.
It removes an N-x multiplier that would otherwise appear the moment large objects are
served.

## Files

| file | change |
|---|---|
| `coord/order/service.go` | parallel signing in `CreateGetOrderLimits` + concurrency knob |
| `coord/file/service.go` | cached file read in `AuthorizeFileOwner`; contract cache |
| `coord/file/endpoint.go` | `lazy_segments` handling; `GetSegmentDownloadInfo`; per-segment accounting |
| `pkg/pb/coord/file/v1/file.proto` | new RPC + `lazy_segments` field (regenerate) |
| `go-sdk/download.go` | lazy segment fetch + one-segment prefetch |
| `internal/testplanet/download_test.go` | multi-segment lazy e2e |

## Verification

**Unit** (`go test ./coord/... ./edgeserver/ -race`)
- Parallel signing returns limits in piece order, identical to the sequential result for
  the same inputs; a signing error still fails the whole call; `-race` clean.
- `AuthorizeFileOwner` and `LoadDownloadData` together perform **one** file read (assert
  on the existing `fakeStore.getCalls` counter).
- Lazy manifest: first covering segment has limits, the rest carry metadata only; without
  `lazy_segments` every segment still has limits (compatibility).
- `GetSegmentDownloadInfo` rejects a ticket that does not authorize the file, and a
  segment number outside the object.
- Accounting: a lazy download bills each segment once, and the sum over lazily-fetched
  segments equals what the eager path bills for the same object.

**End-to-end** (`DEPIN_TEST_POSTGRES=... go test ./internal/testplanet/ -race`)
- Multi-segment file (set `DefaultSegmentSize` below the test data size) downloads
  byte-identical via the lazy path and via the eager path.
- Regression: `TestDownload`, `TestRangedDownload`, `TestDownloadResilience` (run several
  times - it is load-order sensitive and has already caught one probabilistic bug),
  `TestEdge*`, `TestDeleteContract`, `TestDownloadResolveAdmissionControl`.

**Live, after each step separately** - deploy 1, measure, then 2-3, then 4:
- `function_times{role="api",name="(*Service).CreateGetOrderLimits",field="ravg"}` should
  fall with the cap, then again with parallel signing once cores are added.
- `(*Endpoint).GetDownloadInfoByTicket` ravg is the headline number: 13.0s today.
- `edge_download_duration` vs `edge_sdk_fetch_duration` - the gap between them is the
  resolve, and is what this plan attacks.
- Throughput: `40 / service time` should rise; `edge_download_rejected` should fall
  without touching `MaxConcurrentDownloads`.

## Risks

- **Parallel signing is inert on one core**, and will look like a no-op if measured before
  cores are added. Measure it only after.
- **Per-segment accounting changes billed numbers.** Under-billing a client who abandons a
  download is defensible, but it is a change to invoice inputs and needs sign-off.
- **Lazy minting adds a round trip per segment** unless the prefetch lands. Ship both
  together, and verify on a multi-segment file that total download time does not regress.
- **Proto compatibility.** The `lazy_segments` opt-in must default false, and the
  coordinator must keep serving eager manifests for as long as any deployed SDK or edge
  predates the change.
- The coordinator DB was down (disk full, `No space left on device` first seen 15:49 UTC
  2026-08-13, then `could not write init file`) when this plan was written. None of this
  is measurable in production until that is resolved.

## Already shipped in this workstream (uncommitted)

Context for whoever executes this - these landed before the plan above:

- Batched worker-bandwidth rollups: `order.AllocationStore` reduced to one method,
  `AddWorkerBandwidthAllocatedBatch`, so a manifest costs one multi-row upsert instead of
  ~35 sequential ones. Measured 35 x 1.25s -> 1 x 0.072s.
- `internal/grpcutil/cache` no longer runs its `Close` callback under the global mutex -
  the pool lookup went from 87ms average (max 5.08s) to 0.76ms. depin is the master copy;
  go-sdk syncs from it, so both were patched.
- `DownloadNodes` tail-tolerance cap (see item 1). Trap found: trimming a shuffled
  candidate list is unsafe when `OptimalShares == TotalShares`, because worker liveness
  reaches the overlay via a periodic snapshot and a dropped candidate may be the live one.
  Skipped for schemes with no real over-provisioning.
- Coordinator admission control: per-file and per-contract rate limiters plus a
  server-side resolve deadline, with the edge mapping `ResourceExhausted` to 503 +
  `Retry-After`. Note **Storj has no global limit** - every limiter is keyed
  (`limiterKey = ProjectID [+ "-get"]`), and its only semaphores are background workers.
