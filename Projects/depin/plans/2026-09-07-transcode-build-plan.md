---
type: plan
project: depin
created: 2026-09-07
tags: [depin, transcode, vod, hls, worker, plan, build-plan]
---

# Build plan: transcode-as-a-service on depin

Derived from the design spec at
`~/personal/tuan/Projects/depin/plans/2026-08-28-transcode-as-a-service-on-depin.md`.
That document is a **design spec** and explicitly says so (`| Deliverable | Design spec only |`).
This is the executable build plan it lacked.

## Context

AIOZ wants VOD transcoding served by the depin worker fleet, so the same nodes
that store data also earn by transcoding it. depin has **zero media code today**:
`feat/transcoding` is 0 commits ahead of `develop`, and the only `transcod` hit in
the repo is the unused `CONTRACT_TYPE_TRANSCODING = 2` enum value at
`pkg/pb/coord/contract/v1/contract.proto:12`.

The design is already settled and is not re-opened here: **depin provides
transcode compute only.** The client probes, plans the ladder, segments the
source, submits segments, collects renditions and writes all manifests. depin
holds the config, advertises and selects capable workers, authorizes and budgets
the work through the existing `OrderLimit` machinery, transcodes a submitted
segment, and meters what was consumed. Segmentation, stitching and packaging are
out of scope, permanently.

Scope of *this plan*: **depin server side only** - protos, coordinator, worker,
and in-repo testplanet e2e coverage. The `go-sdk` client and the client-side
tier-2 perceptual verification it would host are follow-up work, because
`go.mod:415` pins go-sdk to a remote pseudo-version
(`gitlab.internal/aioz-depin/go-sdk v0.0.0-20260821093526-73f0c7097e45`) rather
than a local `../go-sdk` replace, making SDK changes a separate repo and a
coordinated release.

Intended outcome: a flag-gated, metered transcode subsystem that a client can
drive end-to-end, shipped default-off like every prior depin subsystem.

## Decisions taken for this plan

| Question | Answer |
|---|---|
| hub overlap | **Reuse hub's ideas and logic where useful, but all new code is written in the depin source tree.** No dependency on the hub repo, no blocking gate. |
| Scope | depin server-side only; go-sdk is follow-up |
| Schema shape | **Real tables.** transcode gets its own config tables; the dropped `storage_configs` precedent does not apply. |
| ffmpeg | **Subprocess** (`exec.Command`), not cgo/libav bindings |

### Why ffmpeg is a subprocess

`aioz-stream-core` shells out (`internal/utils/ffprobe/ffprobe.go:142`,
`internal/core/general_hanlder.go:351`) and carries no cgo or LPMS bindings. Match
it. CI's `test:compile-check` job cross-compiles with `CGO_ENABLED=0 -tags purego`
for `linux/arm64 windows/amd64 freebsd/amd64 darwin/arm64`; a cgo libav binding
breaks that matrix and complicates worker packaging. Subprocess also lets the
worker advertise **measured** capability from a boot probe of whatever ffmpeg the
host actually has.

---

## Phase 0 - Mine hub for reusable logic (non-blocking)

`hub` (`gitlab.internal:dai.trong.cao/hub`, `feat/transcoding`) is a teammate's
greenfield DePIN rewrite that already does segment-first fan-out, with its own
scene detection, orchestration, assembly and VMAF scoring. **Treat it as a source
to learn from and port from, not as a dependency.** Every line that ships lands in
the depin tree; depin does not import, vendor or call hub.

This phase runs in parallel with Phase 1 and feeds Phases 4 and 6. Read hub's
transcode path and write down, per component, whether depin ports the logic,
re-derives it, or skips it. Where hub's logic is sound, porting it beats
rediscovering it.

Three hub properties are known **anti**-patterns from the spec's study - do not
carry them across:

- NAT traversal by SSH reverse tunnel (`cmd/tunnel`), with a hardcoded OpenSSH
  private key committed at
  `internal/worker/client/infrastructure/rest/rest.go:654`. depin already has mTLS
  plus relay transport and needs none of it.
- Worker selection is "any online worker with >100MB free disk, in address order"
  (`worker_task.go:107-187`) - no scoring, no load, no capability match.
- `libvmaf` runs **on the worker** and the reported score is trusted
  (`step_transcode_score.go:181-269`). depin's billable quantity is fixed in a
  coordinator-signed limit before the work starts.

Its `extractAudio` path (`internal/transcode/usecase/contract.go:665`) is worth
reading: hub independently reached the same conclusion as this design, that audio
must not be chunked.

### Schema: transcode gets real tables

Decided, not open. `transcode_configs` / `transcode_renditions` /
`transcode_sessions` are real tables.

Worth knowing while writing them, since the design spec cites it as precedent:
`storage_configs` no longer exists.
`coord/db/migrations/000029_drop_storage_config.up.sql` dropped it and
`contracts.storage_config_id` ("Segment size + the default placement now live in
coordinator config (file.Config)"), and the proto still carries the scar -
`reserved 11; // was storage_config_id (removed with StorageConfig)`. That
rationale does not transfer: segment size and placement are a handful of global
choices, whereas a rendition ladder is per-client, arbitrary-cardinality data that
sessions must reference by id and workers must resolve at encode time. Follow the
conventions of `000002_init_client.up.sql`, not the fate of `storage_configs`.

---

## Phase 1 - Protos

All protos live under `pkg/pb/`, package `hub.<subsystem>.v1`, `option go_package
= "aioz-depin/pkg/pb"`, gogoproto annotations, regenerated with `make proto`
(`Makefile.build:345`, driven by root `buf.yaml` / `buf.gen.yaml`:
`protoc-gen-go-grpc` + `protoc-gen-gogofaster`).

Field numbers verified against the current tree - `contact.proto:47-49` documents
the never-renumber-a-deployed-field rule, learned when `signed_tags` took field 7.

| File | Change | Next free field |
|---|---|---|
| `pkg/pb/shared/pieces.proto` | `TRANSCODE = 8` in `PieceAction` | last is `PUT_GRACEFUL_EXIT = 7` |
| `pkg/pb/coord/contact/v1/contact.proto` | `TranscodeCapability transcode = 9` in `CheckInRequest` | last is `exiting = 8` |
| `pkg/pb/shared/worker/info/v1/worker.proto` | `used_for_transcode = 12` in `DiskSpace`; new `TranscodeCapability` message | `DiskSpace` last is `model = 11` |
| `pkg/pb/coord/contract/v1/contract.proto` | transcode config/session messages (append at 13+) | `Contract` last is `placement_id = 12` |
| `pkg/pb/coord/transcode/v1/` | **new**: config CRUD, `OpenSession`, `CloseSession`, `GetTranscodeWorkers`, `ReportDispute` | - |
| `pkg/pb/worker/transcode/v1/` | **new**: `Submit`, `GetStatus`, `Fetch`, `Ack` | - |

`used_for_transcode` is a **reporting breakdown line only, not a second budget** -
see Phase 5.

Worker service surface, mirroring the spec:

```proto
rpc Submit(stream SubmitRequest) returns (SubmitResponse);   // header frame + chunks -> task_id
rpc GetStatus(GetStatusRequest) returns (GetStatusResponse); // batch of task ids
rpc Fetch(FetchRequest) returns (stream FetchResponse);      // one rendition out
rpc Ack(AckRequest) returns (AckResponse);
```

`TranscodeCapability`: available encoders, hwaccel modes, max concurrent tasks,
max resolution. No free-disk field - `DiskSpace` already carries it and transcode
draws on the same allocation.

**Verify:** `make proto` is clean, `go build ./...` passes, `buf` lint/breaking
checks pass in CI (`.gitlab/ci/lint.yml`).

---

## Phase 2 - Coordinator: config + session

New package `coord/transcode/`, following the `coord/payout/` layout exactly:
`config.go` (cfgstruct `Config` + `DefaultConfig()`), `endpoint.go` (embeds
`transcodepb.UnimplementedTranscodeServiceServer`), `monkit.go` (`var mon =
monkit.Package()`), `README.md`.

1. **Migration.** `make coord/create NAME=transcode_configs` - next number is
   **000055** (`000054_period_close_status` is current HEAD; a merge can collide
   here, so re-check before writing). `transcode_configs` (contract_id, name,
   segment duration `D`, immutable-after-first-use), `transcode_renditions`
   (codec, container, w, h, fps, bitrate, crf, profile, level, kind), and
   `transcode_sessions` (transcode_config_id, status, created_at, closed_at).
   Follow the conventions in `000002_init_client.up.sql`: UUID PKs, FK to
   `contracts(id)` `ON DELETE CASCADE`, partial indexes guarded on
   `deleted_at IS NULL`.

   `transcode_sessions` also carries `rendition_indexes` (JSONB): a profile is a
   SUPERSET ladder and a session selects the subset its source justifies, so one
   named profile serves a 4K master and a phone clip without a profile per
   combination. Selected rungs KEEP their profile index - renumbering [2,3,5] to
   [0,1,2] would make `Fetch(task, 0)` mean a different rung in different
   sessions. The compute budget sums over the selected rungs only.

   A `transcode_config` is the long-lived reusable profile ("standard-1080p");
   a `transcode_session` is the per-video instance. Segments are submitted against
   a session, order limits are minted for it, metering aggregates to it - so a
   segment request carries only `session_id` + `segment_index` + bytes and never
   resends the ladder, which matters at 4s segments where a 2-hour video is ~1,800
   submissions. **A live stream is a session that stays open**, so the live path
   needs no new concept.
2. **Repository.** `coord/db/transcode_repo.go` implementing the package's `Store`
   interface on the shared `*gorm.DB`, registered on `UnitOfWork`
   (`coord/db/uow.go`). Follow `coord/db/billing_repo.go`.
3. **Endpoint.** Config CRUD, `OpenSession`, `CloseSession`. Owner checks against
   `contracts.owner`, type must be `TRANSCODING`.
4. **Immutability.** A `transcode_config` referenced by any session must be
   immutable (spec risk #5: a config edited mid-session silently changes output,
   and the worker caches it). Enforce in the repository, not only in the endpoint.
5. **Wiring.** `coord/peer.go`, following `setupPayoutEndpoint` (~line 1341):
   ```go
   b.Server.RegisterGRPCServiceFunc(func(s grpc.ServiceRegistrar) {
       transcodepb.RegisterTranscodeServiceServer(s, endpoint)
   })
   ```
   Mount `Transcode transcode.Config` on the top-level `coord/config.go` struct,
   with `Enabled bool ... default:"false"` gating construction entirely - the
   `if !b.Config.Audit.Enabled { return nil, nil, nil }` pattern at
   `coord/peer.go:1060`.

**Verify:** migration up/down round-trips against the dev Postgres
(`dev/coord/docker-compose.yml`); repository unit tests via `coorddbtest`;
`DeleteContract`'s soft-delete leaves transcode usage history intact, the way
storage usage survives (spec verification item 4).

---

## Phase 3 - Order limits: mint and settle TRANSCODE

The single largest reuse in the design. Every worker resource in depin is already
gated by a coordinator-signed capability token, and it already means what a
transcode authorization needs to mean.

1. **Mint.** Add `NewSignerTranscode(...)` to `coord/order/signer.go` beside
   `NewSignerGet`/`NewSignerPut`/`NewSignerAudit`, and a
   `CreateTranscodeOrderLimits(ctx, workers []*contact.SelectedWorker, session, ...)`
   in `coord/order/service.go` modelled on `CreatePutOrderLimits` (line 167). All
   of it funnels through the existing `Signer.Sign()` (line ~230), the one real
   signing site.
2. **`limit` is the compute-unit budget** for the segment -
   `Σ over renditions of w × h × fps × duration`, computed **coordinator-side from
   the session's config**. Never a worker-reported number. This is the whole
   defence against go-livepeer's central flaw, where the orchestrator bills from a
   `Pixels` value the worker puts in an HTTP header (`server/ot_rpc.go:409-417`)
   and never re-derives it.
3. **Serial per submission.** `serial_number` is the replay key, not `piece_id`;
   `usedserials` keys on coordinator + serial. Mint one per segment submission.
4. **`piece_id` carries the segment content hash.** It binds the authorization to
   exact bytes and doubles as the idempotency key of Phase 5. This repurposes a
   `vo.PieceID`-typed field - **this is spec risk #2 and needs an explicit review
   decision in this phase**, the alternative being a `TranscodeLimitMetadata`
   message alongside `OrderLimitMetadata`
   (`pkg/pb/coord/orders/v1/ordersmeta.proto:19-24`).
5. **Settle.** `coord/order/settlement.go` gates on a `paidActions` whitelist
   (line 238: `GET, PUT, GET_AUDIT, GET_REPAIR, PUT_REPAIR`) and `verifyOrder`
   rejects anything outside it with "limit action %s is not settleable" (line
   258). **`TRANSCODE` must be added to that slice or every transcode settlement
   silently fails.** The spec does not mention this. Update the slice's doc
   comment too - it currently enumerates why each action is there.

**Verify:** unit tests for the mint path; a settlement test that walks mint →
worker-side verify → `SettlementWithWindow` → serial lands in `usedserials` → a
replay is rejected (spec verification item 2). Then walk the **retry** case and
confirm the Phase 5 idempotency key prevents a second charge.

---

## Phase 4 - Capability advertisement and selection

1. **Worker declares.** Populate `TranscodeCapability` in the check-in request
   built at `worker/contact/service.go:213-224` (`pingCoordOnce`), sourced from
   `UpdateSelf` (line 284) the way `SystemInfo`/`DiskSpace` already are.
2. **Measure, never trust the declaration.** Run a boot-time probe transcode and
   advertise the *measured* result. aioz-stream-core's `GpuConfig.Config` is
   `gorm:"-"`, so its capability flags are never persisted, `selectGPU` returns nil
   and everything silently runs on CPU. Do not reproduce that.
3. **Coordinator stores it.** Persist on the workers table via the check-in
   handler; extend `coord/contact/selectedworker.go` (a 51-line struct that today
   carries `FreeDisk` as its only capacity dimension) with the transcode
   dimensions.
4. **Selector.** New `coord/overlay/transcodeselection.go` beside
   `uploadselection.go` / `downloadselection.go`, with a periodic cache in the same
   shape (`NewUploadSelectionCache` / `Refresh` / `Run` / `GetWorkers`).
   Capability-matched **and load-aware** - hub's selector has no scoring, no load,
   no capability match, and that is the failure to avoid.
5. **Endpoint.** `GetTranscodeWorkers(session_id, n)` on the coord transcode
   endpoint, returning `{worker addr, signed OrderLimit}` pairs by combining
   Phase 3 and this selector.

This partially lands **worker-tags Phase 2** (selection consumption), deferred when
tags shipped 2026-08-04.

**Verify:** selector unit tests with synthetic worker sets (capability mismatch,
at-capacity, low-disk); a testplanet check that a worker's declared capability
reaches the coordinator and back out through `GetTranscodeWorkers`.

---

## Phase 5 - Worker transcode service

New package `worker/transcode/`, following `worker/piecestore/` - the only worker
package that owns a gRPC endpoint and the order-limit verification path.

Files: `service.go` (config + business logic), `endpoint.go` (the four RPCs),
`store.go` (task store), `executor.go` (bounded encode pool), `gc.go` (reap chore),
`sessioncache.go`, `monkit.go`.

### 5.1 Task lifecycle

```
ACCEPTED --> RUNNING --> DONE --> (Ack | TTL) --> REAPED
                 +-----> FAILED{reason, retryable}
```

- **`Submit` returns as soon as bytes are accepted and the limit validates.** It
  does not wait for the encode. No RPC is held open across it.
- **Idempotency is load-bearing, not a nicety** (spec risk #3). A network failure
  after the worker accepted but before the client saw `task_id` would otherwise
  double-encode and double-bill, because a retry needs a fresh order limit -
  serials are single-use. Key on `(session_id, segment_index, content_hash)`; a
  repeat returns the existing `task_id` and does **not** consume the new limit.
- **Backpressure is explicit.** At capacity, `Submit` rejects with a typed
  `AT_CAPACITY` so the client immediately picks another worker. Silent queueing is
  what turns a slow worker into a stalled ladder.
- **Settlement happens on `DONE`**, not on fetch. The work was performed whether or
  not the client collects it.

### 5.2 Order-limit verification

Reuse the worker's existing sequence verbatim -
`worker/piecestore/verification.go:25-120` (`verifyOrderLimit`): action allowed →
addressed to this worker (`s.identity.ID.Equal(limit.WorkerId)`) → required fields
present → not expired → `OrderLimitGracePeriod` window → coordinator signee
resolved through `s.trust.GetSignee` (fail-closed) →
`s.signatureCheck.VerifyOrderLimitSignature` →
`s.usedSerials.Add(...)` replay check. Then transcode-specific gates: declared
compute units within `limit`, capacity available, output-disk budget available.

Extend `actionAllowed`'s expected-action sets for `TRANSCODE`.

### 5.3 Streaming

Model `Submit` on the piecestore `Upload` loop and `Fetch` on the `Download` loop
(`worker/piecestore/endpoint.go`, `const downloadChunkSize = 64 * memory.KiB` at
line 33). Keep the strict-ordering check `msg.Chunk.Offset != writer.Size()`. The
`sendDownloadTrailer` pattern (~line 1147) is the template for `Fetch`'s final
frame.

### 5.4 Task store

Worker-local SQLite via GORM, one named sub-DB. Follow `worker/db/`: add
`"transcode"` to `dbNames` in `openDatabases` (`worker/db/database.go:53`), a
`Container` implementation, a `TranscodeDB()` accessor on `WorkerDB` mirroring
`CoordDB()` (line 33), and a repository whose constructor calls `AutoMigrate` -
GORM AutoMigrate is the real source of truth here; `worker/db/migrations/*.sql`
are empty placeholders (`worker/db/README.md:29-33`). `PieceExpirationRepository`
(`worker/db/piece_repos.go:25-36`) is the template for the batched "list due / GC"
queries.

### 5.5 Disk

**Output shares the worker's existing storage allocation. One disk, one budget.**
`SharedDisk.DiskSpace` (`worker/pkg/space/shared.go:60-103`) already splits one
`allocated` pool into `UsedForPieces` / `UsedForTrash` / `UsedReclaimable` /
`Reserved`; transcode adds one more breakdown line, reported as
`used_for_transcode` (proto field 12) and subtracted from `available`.

The consequence is intentional: transcode output reduces `free`/`available`, and
`coord/contact/selectedworker.go` already feeds `FreeDisk` into upload selection,
so a busy transcoder automatically receives fewer piece uploads. Transcode and
storage back-pressure each other through accounting that already exists.

That puts all the weight on the worker's admission check. **This is spec risk #1
and the plan's single most important default**: cap pending transcode output as a
**fraction of currently free space**, not an absolute figure. It needs a real
default and a monkit metric, not a TODO. Without it a burst of large submissions
consumes the allocation the piece store needs.

### 5.6 GC chore and wiring

Reap expired tasks; `Ack` is the normal path, TTL the safety net. Register on
`peer.Services` exactly like the used-serials chore (`worker/peer.go:582-594`):

```go
peer.Services.Add(lifecycle.Item{
    Name:  "transcode:gc",
    Run:   peer.Transcode.GCChore.Run,
    Close: peer.Transcode.GCChore.Close,
})
```

Add a `Transcode struct{...}` group to the `Peer` struct (`worker/peer.go:103-150`)
alongside `Contact` / `Storage` / `Console`, register the service on the existing
mTLS server the way `piecepb.RegisterPieceStoreServiceServer(peer.Server.GRPC(),
...)` does at line 617, and gate the whole construction on
`config.Transcode.Enabled` (default `false`) following `config.Console.Enabled` at
line 636 - when off, nothing is constructed, no goroutine, no lifecycle item.

Reusing the existing transport is deliberate: mTLS identity, relay-based NAT
traversal and connection pooling come free, and the client always dials, so
polling works through NAT with no tunnel.

### 5.7 Session cache

The worker resolves the ladder and segment duration `D` from `session_id` (cached
from the coordinator, refreshed on miss) rather than trusting them in the request.
The config is what was authorized and billed against.

**Verify:** unit tests on the endpoint's validation logic constructed directly,
without gRPC, the way `worker/piecestore/verification_test.go` does
(`newVerifyService`, `TestVerifyOrderLimitSerialIsSingleUse`). Then walk the task
state machine for four failure modes (spec verification item 3): client never
polls, client never fetches, worker restarts mid-encode, worker restarts after
DONE. Confirm disk is always reclaimed and settlement is never double-counted.

---

## Phase 6 - Encode core

`worker/transcode/ffmpeg/`: arg builder, ladder resolution, probe, error
classification. Written fresh, but these techniques are proven in
`aioz-stream-core` and must be reproduced rather than rediscovered.

1. **Cross-rendition IDR alignment** - what makes an ABR ladder switchable.
   Expression-based, not GOP-count, so VFR and unknown-fps sources still align
   (`args.go:595-600`):
   ```
   -force_key_frames expr:gte(t,n_forced*D)   -fps_mode cfr
   ```
   `D` comes from the session's config, which is why it belongs on the contract.
2. **Resolution resolution** - drive the rung's longer edge onto the source's
   longer edge (portrait falls out for free), SAR-corrected, rotation-aware,
   clamped to NVENC limits, forced even, resolved *numerically* so the client can
   advertise a true `RESOLUTION` (`quality_config.go:192-260`).
3. **GPU fallback chain** `AccelFull → AccelEncodeOnly → AccelNone`
   (`worker.go:529-592`). The middle mode exists because NVDEC and NVENC fail
   independently.
4. **Non-retryable error classification**, ported from LPMS
   (`ffmpeg_errors.go:54-78` → `transcode.go:124-176`), so a bad submission fails
   fast with `retryable=false` instead of burning the budget across three workers.
5. **Pre-flight probe** - ffprobe plus a keyframe check before spending encode
   time.
6. **HEVC on Apple**: `-tag:v hvc1` (not `hev1`) and fMP4, or Safari/iOS refuses
   the stream.

Spec risk #4 is that writing this fresh re-earns aioz-stream-core's edge cases.
**Budget for the fixture corpus, not the happy path.** Its 14-file corpus
(`internal/core/worker/fixtures_test.go:65-190` - rotated, anamorphic, 10-bit,
HDR, VFR, 4:2:2, RGB, audio-only, no-audio, 120fps) is the input-robustness
checklist to reproduce even though the code is not.

Tests that need ffmpeg on PATH must skip cleanly when it is absent, or CI's `test`
job breaks for everyone.

**Verify - do this first, before anything else in the phase.** Spec verification
item 1, and if it fails the client contract is wrong and everything downstream is
too: transcode one source whole, and the same source pre-split into 3 segments
with identical `-force_key_frames` expressions; confirm the concatenated output is
frame-identical and segment durations match at the seams.

---

## Phase 7 - Metering and observability

1. Add `TRANSCODE` to the settlement counters in `coord/order/settlement.go`
   (already tagged by action, emitting `order_settlement_bytes` /
   `order_settlement_rows`) and to the worker tally read path.
2. Aggregate per session for client-facing usage.
3. `monkit` metrics: tasks accepted / running / done / failed by reason,
   `AT_CAPACITY` rejections, queue depth, encode duration, output bytes held, and
   **the fraction-of-free-disk headroom from 5.5**.
4. `ReportDispute` receiver on the coord endpoint plus worker deprioritization in
   the Phase 4 selector. The client-side detection that files those disputes is
   follow-up work.

Explicitly **not** wired into invoices, reservations or payouts - the deliberate
consequence of the metering-only decision, and what lets the economics be set once
real cost data exists.

**Verify:** metrics appear in the debug endpoint under a testplanet run; a settled
transcode order shows up in the worker tally.

---

## Phase 8 - End-to-end and ship

1. **testplanet e2e** in `internal/testplanet/transcode_test.go`, following
   `worker_stored_bytes_test.go:27`. Needs a minimal in-repo test client for
   submit/poll/fetch (go-sdk is out of scope), a `Reconfigure.Coord` /
   `Reconfigure.Worker` hook pair to flip `Transcode.Enabled` on, and ffmpeg on
   PATH with a clean skip when absent.
   Remember `DEPIN_TEST_POSTGRES` - without it `testplanet.Run` calls `t.Skip` and
   the test silently does nothing (`internal/testplanet/run.go:33-35`). Use the
   `depin-pgtest` instance on :15445, never the coord-db on :5445.
2. **Sizing exercise** (spec verification item 5): at 4s segments and a 5-rung
   1080p ladder, how much of `allocated` does a worker hold at N concurrent
   sessions with a T-second TTL, and what fraction-of-free cap keeps the piece
   store healthy? This sets the Phase 5.5 default.
3. **Full suite green**: `go test -tags purego -race -vet=off -timeout 30m ./...`,
   `make lint`, and the `test:compile-check` cross-compile matrix.
4. **Ship default-off.** Both `Config.Transcode.Enabled` flags stay `false`.

---

## Follow-ups (explicitly not this plan)

- **go-sdk client** - config/session APIs, submit/poll/fetch, worker retry,
  tier-1 checks. Needs a coordinated release because `go.mod:415` pins go-sdk to a
  remote pseudo-version.
- **Tier-2 sampled redundancy.** Client-side, because the client holds the bytes
  and is the party motivated to detect bad output. Encoders are non-deterministic,
  so byte comparison is useless: compare **MPEG-7 video signatures** (ffmpeg's
  `signature` filter) as go-livepeer's fast verification does
  (`server/broadcast.go:713-747`), escalating to full frame comparison on
  mismatch. Two traps not to repeat: go-livepeer compares only one randomly chosen
  rendition (`broadcast.go:686`) - a fine strategy, but state it rather than imply
  full coverage; and its `Policy.SampleRate` / `Policy.Redundancy` are declared and
  **never read** (`verification/verify.go:87-90`), so "sampled" verification
  silently runs on every segment. Wire the knob or do not ship it.
- **The published client contract** - segments independently decodable; segment
  duration an integer multiple of `D`; **do not chunk audio** (encoder delay and
  priming samples cause audible discontinuities at every boundary - declare audio
  as a whole-source rendition kind); pipeline submissions and poll in batches;
  `Ack` after fetching; client renumbers and writes all manifests, skipping failed
  renditions so a partial ladder still ships.
- **Live.** A live stream is a session that stays open - the async submit/poll/
  fetch shape already fits. Later additions only: sticky selection, hard deadlines
  (drop a late segment, do not retry), tier-2 inline or skipped, shorter TTL and
  eager `Ack`. Do **not** adopt go-livepeer's ingest segmenter, which waits for
  segment *N+1*'s file to appear before releasing segment *N*
  (`lpms:segmenter/video_segmenter.go:218-219`) - a structural one-segment latency
  floor.

## Risk register

| # | Risk | Phase | Mitigation |
|---|---|---|---|
| 1 | hub duplicates this work | 0 | Parallel effort, accepted. Port hub's logic where it is sound; ship it in the depin tree |
| 2 | `TRANSCODE` missing from `paidActions` → silent settlement failure | 3 | Explicit task; test the whole round trip |
| 3 | `piece_id` repurposed for a content hash | 3 | Review decision vs a metadata message |
| 4 | Double-billing on retry | 5 | Idempotency key; test the retry path |
| 5 | Transcode output starves the piece store | 5 | Fraction-of-free cap with a real default and a metric |
| 6 | Encode core re-earns aioz-stream-core's edge cases | 6 | Port the 14-file fixture corpus |
| 7 | Session config mutated mid-session | 2 | Immutability enforced in the repository |
| 8 | ffmpeg absent in CI | 6, 8 | Clean skips; subprocess keeps `CGO_ENABLED=0` green |
| 9 | Migration number collision on merge | 2 | Re-check `ls coord/db/migrations \| tail` before writing |
