---
type: plan
project: depin
created: 2026-08-28
tags: [depin, transcode, vod, hls, worker, plan]
---

# Design spec: transcode-as-a-service on depin

## Context

AIOZ wants VOD transcoding (and later live ABR) served by the depin worker fleet,
so the same nodes that store data also earn by transcoding it.

**depin has zero media code today.** An exhaustive grep for
`ffmpeg|transcod|codec|hls|m3u8|VOD` returns only comments citing HLS as a
motivating workload (`coord/file/service.go:42`, `edgeserver/readahead.go:28`).
The worker's whole RPC surface is `Upload`, `Download`, `GetChunkProof`,
`GetSignedPiece`, `Retain`, `RestoreTrash`
(`pkg/pb/worker/piece/v1/piece.proto:13-36`) plus `Ping`. The only tasks ever
pushed to a worker are GC and audit reads.

**But the contract model already anticipates this.**
`pkg/pb/coord/contract/v1/contract.proto:10-14` declares:

```proto
enum ContractType { CONTRACT_TYPE_UNSPECIFIED = 0; STORAGE = 1; TRANSCODING = 2; }
```

`TRANSCODING` is already reserved and unused. The `contracts` table carries only
shared fields (`coord/db/migrations/000002_init_client.up.sql:26-42`: id, owner,
type, status, timestamps) with per-type config in a **sibling table**
(`storage_configs`, `:113-125`), and the contract proto carries
`reserved 11; // was storage_config_id (removed with StorageConfig)` - they
deliberately moved to config-points-at-contract. This design fills the slot left
open.

Two adjacent systems were studied:

- **`aioz-stream-core`** (`/home/tuan/work/stream/aioz-stream-core`) - the
  existing centralized transcoder. Mature ffmpeg layer; one worker per whole
  video, renditions sequential (`worker.go:387-414`). No livestream.
- **`hub`** (`gitlab.internal:dai.trong.cao/hub`, `feat/transcoding`) - a
  teammate's greenfield DePIN rewrite that already does segment-first fan-out,
  with its own scene detection, orchestration, assembly and VMAF scoring.

**`go-livepeer-openpool`** was studied in depth and is the closest match to what
we want: its gateway segments and assembles; the orchestrator/transcoder does
nothing but transcode a segment it is handed. That is the split adopted here.

## Scope

**depin provides transcode compute. Nothing else.**

| depin does | Client does |
|---|---|
| Hold the transcode config on a contract | Probe the source |
| Advertise worker transcode capability | Plan the ladder |
| Select capable, available workers | **Segment the source** |
| Authorize + budget the work (order limits) | Submit segments, collect renditions |
| Transcode a submitted segment | **Assemble HLS/DASH manifests** |
| Verify honesty by sampling | Store outputs wherever it wants |
| Meter what was consumed | |

Segmentation, keyframe planning, ladder selection, stitching and packaging are
**explicitly client-side** and out of scope for depin.

## Decisions taken

| Decision | Choice |
|---|---|
| Target repo | **`depin`** |
| Segmentation | **Client-side. depin transcodes segments only.** |
| Config | **On the contract**, applied to many segments |
| Config scope | **Both** - reusable profile + per-video session |
| Input path | Client pushes segment bytes directly to the worker |
| Execution | **Asynchronous. No held stream during the encode.** |
| Output store | **Worker-local under TTL; client pulls by task id** |
| Output disk | **Shares the worker's existing storage allocation** |
| Completion | **Batch polling** |
| Manifests | Client, entirely |
| Worker fleet | Same fleet, new capability |
| Encode core | Write fresh in depin |
| Trust model | Cheap checks always + sampled redundancy |
| Payments | Metering only, no payout wiring |
| Livestream | Design for it, build VOD first |
| Deliverable | **Design spec only** |

---

## Architecture

### 1. Contract model - shared fields, sibling config table

Follow the pattern the schema already uses, exactly as `storage_configs` does.

```
contracts                 (EXISTING, unchanged)
  id, owner, type=TRANSCODING, status, placement_id, timestamps
    │
    ├── storage_configs           (EXISTING)  contract_id, region
    │
    └── transcode_configs         (NEW)  the reusable ladder profile
          id, contract_id, name, segment_duration D, immutable_after_first_use
            │
            ├── transcode_renditions  (NEW)
            │     codec, container, width, height, fps, bitrate, crf,
            │     profile, level, kind (video|audio)
            │
            └── transcode_sessions    (NEW)  one per source video / live stream
                  id, transcode_config_id, status, created_at, closed_at
```

- A **`transcode_config` is the long-lived reusable profile** ("standard-1080p").
  Many videos submit against it.
- A **`transcode_session` is the per-video instance.** Segments are submitted
  against it, order limits are minted for it, metering aggregates to it.

Segment requests therefore carry only `session_id` + `segment_index` + bytes. The
ladder is never resent, which matters at 4s segments: a 2-hour video is ~1,800
submissions.

**A live stream is a session that stays open.** No new concept needed later.

`contracts.type` already distinguishes these, so `DeleteContract`, contract tags,
usage-by-tag and billing keep working unchanged.

### 2. Flow

```
setup (once)      client ──CreateContract(TRANSCODING) + CreateTranscodeConfig(ladder, D)──► coord
per video         client ──OpenSession(config_id)──► coord
per batch         client ──GetTranscodeWorkers(session_id, n)──► coord
                  coord  ──[]{worker addr, signed OrderLimit}──► client

per segment  (1)  client ──Submit(OrderLimit, session_id, seg_index, bytes)──► worker
                  worker ──task_id (returns as soon as bytes are accepted)──► client
             (2)  worker   transcodes in background, bounded by capacity
             (3)  client ──GetStatus([]task_id)──► worker        (batch poll)
             (4)  client ──Fetch(task_id, rendition_index)──► worker   (streams bytes out)
             (5)  client ──Ack(task_id)──► worker                (frees disk early)

                  worker ──settle order in the existing settlement window──► coord
                  client   tier-1 checks; with probability p, tier-2 redundancy
                  client ──ReportDispute──► coord   (on mismatch)
per video         client ──CloseSession──► coord
```

No RPC is held open across the encode. `Submit` lives only as long as the upload;
`Fetch` only as long as the download. The coordinator is never on the data path.

### 3. Worker task lifecycle

```
ACCEPTED ──► RUNNING ──► DONE ──► (Ack | TTL) ──► REAPED
                 └────► FAILED{reason, retryable}
```

- **`Submit` returns immediately** once bytes are received and the order limit
  validates. It does not wait for the encode.
- **Idempotency is required, not optional.** A network failure after the worker
  accepted but before the client saw `task_id` would otherwise double-encode and
  double-bill, because a retry needs a fresh order limit (serials are single-use).
  Key on `(session_id, segment_index, content_hash)`; a repeat returns the
  existing `task_id` and does not consume the new limit.
- **Backpressure is explicit.** At capacity, `Submit` rejects with a typed
  `AT_CAPACITY` so the client immediately picks another worker instead of
  queueing. Silent queueing is what turns a slow worker into a stalled ladder.
- **TTL and `Ack`.** Outputs live under a TTL comfortably longer than a realistic
  poll-plus-fetch cycle; `Ack` frees them early. A GC chore reaps expired tasks,
  registered on `peer.Services` as a `lifecycle.Item` like the existing chores.
- **Settlement happens on `DONE`**, not on fetch. The work was performed whether
  or not the client collects it.

**Output disk comes out of the worker's existing storage allocation.** One disk,
one budget. `DiskSpace` already splits a single `allocated` pool into
`used_for_pieces`, `used_for_trash` and `used_reclaimable`
(`pkg/pb/shared/worker/info/v1/worker.proto:68-88`), so transcode output is one
more breakdown line - add `used_for_transcode` as reporting, **not** as a second
budget.

The consequence is intentional and worth stating: transcode output reduces
`free`/`available`, and the coordinator's upload selection already reads
`FreeDisk` (`coord/contact/selectedworker.go:31`), so a busy transcoder
automatically receives fewer piece uploads. Transcode and storage back-pressure
each other through the accounting that already exists, with no new coordinator
concept.

What this puts weight on is the worker's own admission check: cap pending
transcode output as a **fraction of currently free space** rather than an absolute
figure, keep TTLs short, and treat `Ack` as the normal path with TTL as the safety
net. Without that cap, a burst of large submissions can consume the allocation the
piece store needs.

### 4. Authorization: reuse OrderLimit

depin already gates every worker resource with a coordinator-signed capability
token (`pkg/pb/shared/orders.proto:12-52`):

```
OrderLimit { piece_id, coordinator_signature, uplink_public_key, coordinator_id,
             worker_id, limit, action, order_creation, order_expiration,
             piece_expiration, serial_number }
```

It already means what a transcode authorization needs to mean: *this client may
perform `action`, up to `limit`, at `worker_id`, until `order_expiration`, once,
under `serial_number`.*

- Add `TRANSCODE = 8` to `PieceAction` (`pkg/pb/shared/pieces.proto:12-21`).
- `limit` carries the **compute-unit budget** for the segment
  (`Σ over renditions of w × h × fps × duration`), computed coordinator-side from
  the session's config. **Never a worker-reported number.**
- Mint through the existing `Signer.Sign()` (`coord/order/signer.go`) - the single
  real signing site every `Create*OrderLimits` path already shares.
- Settle through the existing `SettlementWithWindow` (`coord/order/settlement.go`).

**Metering therefore falls out of machinery that already exists, is already
signed, and is already instrumented.** This is the largest single reuse here.

Two cautions:
- **`serial_number` is the replay key, not `piece_id`** - `usedserials` keys on
  coordinator + serial. Mint **per segment submission**; reusing one breaks the
  moment two submissions overlap.
- `piece_id` has no natural meaning here. Recommended: carry the **content hash of
  the submitted segment**, which binds the authorization to exact bytes and doubles
  as the idempotency key above. This repurposes a `vo.PieceID`-typed field and
  should be reviewed; the alternative is a `TranscodeLimitMetadata` message
  alongside the existing `OrderLimitMetadata`
  (`pkg/pb/coord/orders/v1/ordersmeta.proto:19-24`).

### 5. Worker service

Register a `TranscodeService` on the worker's existing mTLS gRPC server exactly as
the current services are (`worker/peer.go:435-443`:
`contactpb.RegisterContactServiceServer(peer.Server.GRPC(), endpoint)`), with the
GC chore added to `peer.Services` via `lifecycle.Item`.

```
rpc Submit(stream SubmitRequest) returns (SubmitResponse);        // header + chunks → task_id
rpc GetStatus(GetStatusRequest) returns (GetStatusResponse);      // batch of task ids
rpc Fetch(FetchRequest) returns (stream FetchResponse);           // one rendition out
rpc Ack(AckRequest) returns (AckResponse);
```

Reusing the existing transport is deliberate: mTLS identity, relay-based NAT
traversal and connection pooling come free, and the client always dials, so
polling works through NAT. hub had to build an **SSH reverse tunnel**
(`cmd/tunnel`) to reach NAT'd workers, and ships a **hardcoded OpenSSH private key
committed in source** at
`internal/worker/client/infrastructure/rest/rest.go:654`. We inherit none of that.

The worker resolves the ladder and `D` from `session_id` (cached from the
coordinator, refreshed on miss) rather than trusting them in the request - the
config is what was authorized and billed against.

Gating order on `Submit`: coord signature valid → `worker_id` is me → not expired
→ serial unused (or known idempotency key) → declared compute units within `limit`
→ capacity and output-disk budget available.

### 6. Encode core (fresh, in `worker/transcode/ffmpeg/`)

Written fresh per the decision, but these techniques are proven in
`aioz-stream-core` and should be reproduced, not rediscovered:

- **Cross-rendition IDR alignment** - what makes an ABR ladder switchable.
  Expression-based, not GOP-count, so VFR and unknown-fps sources still align
  (`args.go:595-600`):
  ```
  -force_key_frames expr:gte(t,n_forced*D)   -fps_mode cfr
  ```
  `D` comes from the session's config, which is why it belongs on the contract.
- **Resolution resolution** - drive the rung's longer edge onto the source's
  longer edge (portrait falls out for free), SAR-corrected, rotation-aware,
  clamped to NVENC limits, forced even, resolved *numerically* so the client can
  advertise a true `RESOLUTION` (`quality_config.go:192-260`).
- **GPU fallback chain** `AccelFull → AccelEncodeOnly → AccelNone`
  (`worker.go:529-592`); the middle mode exists because NVDEC and NVENC fail
  independently.
- **Non-retryable error classification** ported from LPMS
  (`ffmpeg_errors.go:54-78` → `transcode.go:124-176`), so a bad submission fails
  fast with `retryable=false` instead of burning the budget across three workers.
- **Pre-flight probe** - ffprobe + keyframe check before spending encode time.
- **HEVC on Apple**: `-tag:v hvc1` (not `hev1`) and fMP4, or Safari/iOS refuses
  the stream; the client needs matching `#EXT-X-VERSION:7` and `CODECS="hvc1.…"`.

Its 14-file fixture corpus (`internal/core/worker/fixtures_test.go:65-190` -
rotated, anamorphic, 10-bit, HDR, VFR, 4:2:2, RGB, audio-only, no-audio, 120fps)
is the input-robustness checklist to reproduce even though the code is not.

### 7. Capability advertisement and selection

Workers already report the needed inventory on every check-in: `SystemInfo`
carries `repeated CPU cpus` and **`repeated GPU gpus`**
(`pkg/pb/shared/worker/info/v1/worker.proto:49-66`). It is telemetry only; nothing
schedules on it.

Add `TranscodeCapability` to `CheckInRequest`
(`pkg/pb/coord/contact/v1/contact.proto:21-51`) - available encoders, hwaccel
modes, max concurrent tasks, max resolution - **appended at field 9**. Free disk
needs no new field; the existing `DiskSpace` already carries it, and transcode
draws on the same allocation. The file documents the "never renumber a deployed field"
rule at lines 46-49, learned when `signed_tags` took field 7.

Extend `coord/contact/selectedworker.go:31` (today only `FreeDisk`) and add a
transcode selector in `coord/overlay/` beside the existing upload/download
selectors. This partially lands **worker-tags Phase 2** (selection consumption),
deferred when tags shipped 2026-08-04.

Two failure modes to avoid, both observed:

- **Verify capabilities, never trust the declaration.** aioz-stream-core's
  `GpuConfig.Config` is `gorm:"-"`, so its capability flags are never persisted;
  `selectGPU` returns nil and everything silently runs on CPU. Run a boot-time
  probe transcode and advertise the *measured* result.
- **Selection must be load-aware.** hub's is "any online worker with >100MB free
  disk, in address order" (`worker_task.go:107-187`) - no scoring, no load, no
  capability match.

### 8. Verification

The client holds the bytes, so verification lives there. This is go-livepeer's
arrangement, and it is sound for the reason that matters: the client is the party
motivated to detect bad output.

**Tier 1 - always, client-side, cheap.** Container and codec match the config,
resolution matches, duration within tolerance, segment count matches `D`, no
zero-length output. Comparable to `core/orchestrator.go:755-776`. Catches
breakage, not cheating.

**Tier 2 - sampled redundancy, the integrity check.** With probability `p`, submit
the same segment to a second worker and compare perceptually. Encoders are
non-deterministic, so byte comparison is useless: compare **MPEG-7 video
signatures** (ffmpeg's `signature` filter), as go-livepeer's fast verification
does (`server/broadcast.go:713-747`), escalating to full frame comparison on
mismatch. Disagreement files a **signed dispute** to the coordinator, which
deprioritizes or suspends the worker in selection.

**Why this avoids go-livepeer's central flaw.** There, the orchestrator bills from
a `Pixels` value the worker puts in an HTTP header
(`server/ot_rpc.go:409-417` → `segment_rpc.go:206`) and never re-derives it, so a
one-line change inflates a worker's payout. Here the billable quantity is fixed in
a **coordinator-signed order limit before the work starts**, computed from the
session's config. A worker cannot inflate it, only fail to earn it.

hub has the same flaw from the other direction: it runs `libvmaf` **on the
worker** and reports the score (`step_transcode_score.go:181-269`).

Two traps not to repeat: go-livepeer compares only one randomly chosen rendition
(`broadcast.go:686`) - a fine sampling strategy, but state it rather than imply
full coverage; and its `Policy.SampleRate` / `Policy.Redundancy` are declared and
**never read** (`verification/verify.go:87-90`), so "sampled" verification
silently runs on every segment. Wire the knob or do not ship it.

### 9. Metering (no payout)

Settled order limits already carry worker, client, action and consumed amount
through `SettlementWithWindow`. Add `TRANSCODE` to the existing settlement
counters (`coord/order/settlement.go`, already tagged by action) and to the worker
tally read path. Aggregate per session for client-facing usage.

Explicitly **not** wired into invoices, reservations or payouts - the deliberate
consequence of the metering-only decision, and what lets the economics be set once
real cost data exists.

### 10. Live path - what this must not preclude

A live stream is a `transcode_session` that stays open. The async submit/poll/
fetch shape still works, and is in fact what live needs. Later additions only:

- **Sticky selection** - consecutive segments to the same worker, avoiding
  per-segment encoder re-init.
- **Hard deadlines** - a late segment is dropped, not retried; `FAILED` rather
  than a growing queue.
- **Tier-2 verification inline or skipped** - a post-hoc perceptual comparison is
  worthless once the segment has been served.
- **Shorter TTL and eager `Ack`** so live output does not accumulate on disk.
- **Do not adopt go-livepeer's ingest segmenter.** It waits for segment *N+1*'s
  file to appear before releasing segment *N*
  (`lpms:segmenter/video_segmenter.go:218-219`) - a structural one-segment latency
  floor. Client-side segmentation should stream, not poll a filesystem.

---

## The client-side contract

Out of scope to build, but depin's API only works if clients honour it. Publish
alongside the SDK:

1. **Segments must be independently decodable** - each begins on a keyframe and
   carries its own headers.
2. **Segment duration must be an integer multiple of the config's `D`**, so forced
   IDRs land on a global grid and renditions stay switchable.
3. **Do not chunk audio.** Encoder delay and priming samples create audible
   discontinuities at every boundary. Declare audio as a whole-source rendition
   kind on the config; it is cheap. hub reaches the same conclusion via its
   separate `extractAudio` path (`internal/transcode/usecase/contract.go:665`).
4. **Pipeline submissions and poll in batches.** Submitting one segment and waiting
   serialises the whole video and wastes the fan-out.
5. **`Ack` after fetching.** TTL is the safety net, not the mechanism.
6. **The client renumbers and writes all manifests**, computing `BANDWIDTH` from
   the rung and `RESOLUTION` from the *returned* dimensions, and should **skip
   failed renditions** so a partial ladder still ships - the behaviour
   `aioz-stream-core` implements at `m3u8_helper/helper.go:109-111`.

---

## Components

| Path | Status | Purpose |
|---|---|---|
| `coord/db/migrations/` | new | `transcode_configs`, `transcode_renditions`, `transcode_sessions`, disputes |
| `pkg/pb/coord/contract/v1/contract.proto` | modify | transcode config messages (type enum already has `TRANSCODING`) |
| `pkg/pb/shared/pieces.proto` | modify | `TRANSCODE = 8` in `PieceAction` |
| `pkg/pb/coord/contact/v1/contact.proto` | modify | `TranscodeCapability` at field 9 |
| `pkg/pb/shared/worker/info/v1/worker.proto` | modify | `used_for_transcode` breakdown line in `DiskSpace` (reporting only, same budget) |
| `pkg/pb/coord/transcode/v1/` | new | config CRUD, `OpenSession`, `GetTranscodeWorkers`, `ReportDispute` |
| `pkg/pb/worker/transcode/v1/` | new | `Submit`, `GetStatus`, `Fetch`, `Ack` |
| `coord/transcode/` | new | config + session usecases, selection endpoint, disputes |
| `coord/overlay/transcodeselection.go` | new | capability + load-aware selection |
| `coord/order/` | modify | mint + settle `TRANSCODE` limits via existing `Signer` |
| `coord/contact/selectedworker.go` | modify | capability dimensions |
| `worker/transcode/` | new | endpoint, task store, executor, capacity gate, GC chore, session cache |
| `worker/transcode/ffmpeg/` | new | encode core: arg builder, ladder, probe, error classes |
| `worker/peer.go` | modify | register the service + GC chore |
| `go-sdk` | modify | config/session APIs, submit/poll/fetch, worker retry, tier-1/tier-2 helpers |

No job table, no chunk planner, no assembler.

## Risks

1. **Transcode output shares the piece storage budget** (§3). This is the chosen
   design, and the back-pressure it creates is deliberate - but it means the
   worker's fraction-of-free-space admission cap is the only thing standing
   between a burst of submissions and a worker that can no longer accept pieces.
   That cap needs a real default and a metric, not a TODO.
2. **`piece_id` repurposing.** Carrying a segment content hash in a
   `vo.PieceID`-typed field works but is a semantic overload; review against a
   dedicated metadata message before committing.
3. **Double-billing on retry.** Serials are single-use, so a resubmit after a lost
   response needs a fresh limit. The idempotency key in §3 is what prevents paying
   twice for one encode - it is load-bearing, not a nicety.
4. **Writing the encode core fresh re-earns aioz-stream-core's edge cases** -
   rotation, anamorphic SAR, 10-bit, HDR tonemapping, VFR, audio-only, NVENC
   clamps. Budget for the fixture corpus, not the happy path.
5. **Session config cache coherency.** Workers resolve the ladder from
   `session_id`; a config edited mid-session would silently change output. Configs
   must be immutable once a session references them.
6. **Overlap with hub.** hub already implements a full version of this. Worth an
   explicit conversation before building.

## Verification

Design-spec deliverable, so this verifies the *design*. Before implementation:

1. **Prove the alignment property by hand.** Transcode one source whole, and the
   same source pre-split into 3 segments with identical `-force_key_frames`
   expressions; confirm the concatenated output is frame-identical and segment
   durations match at the seams. If this fails, the client contract is wrong and
   everything downstream is too.
2. **Walk an `OrderLimit` round trip on paper** for `TRANSCODE`: mint via
   `Signer.Sign()`, verify worker-side, settle through `SettlementWithWindow`,
   confirm the serial lands in `usedserials` and a replay is rejected. Then walk
   the *retry* case and confirm the idempotency key prevents a second charge.
3. **Walk the task state machine** for the four failure modes: client never polls,
   client never fetches, worker restarts mid-encode, worker restarts after DONE.
   Confirm disk is always reclaimed and settlement is never double-counted.
4. **Confirm the contract/config/session schema** against the `storage_configs`
   precedent and `DeleteContract`'s soft-delete semantics, so transcode usage
   history survives contract deletion the way storage usage does.
5. **Size transcode's draw on the shared allocation** against a realistic ladder:
   at 4s segments and a 5-rung 1080p ladder, how much of `allocated` does a worker
   hold at N concurrent sessions with a T-second TTL, and what fraction-of-free
   cap keeps the piece store healthy?
6. **Circulate to the hub owner** for overlap and reuse.
