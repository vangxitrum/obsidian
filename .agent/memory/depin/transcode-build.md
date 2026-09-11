---
type: project
project: depin
created: 2026-09-07
tags: [depin, transcode, vod, worker, coord, build]
---

# Transcode-as-a-service build (depin)

Build plan: `Projects/depin/plans/2026-09-07-transcode-build-plan.md` (approved
2026-09-07). Design spec it derives from: `2026-08-28-transcode-as-a-service-on-depin.md`.
Branch `feat/transcoding`, worktree `.treehouse/depin-b971d9/2/depin`. Uncommitted.

## Decisions that changed the spec

- **hub**: reuse its ideas/logic where sound, but every line ships in the depin
  tree. Not a dependency, not a blocking gate.
- **Scope**: depin server side only. go-sdk is a follow-up because `go.mod:415`
  pins go-sdk to a REMOTE pseudo-version (`gitlab.internal/aioz-depin/go-sdk
  v0.0.0-20260821093526-...`), not a local `../go-sdk` replace -- SDK work is a
  separate repo + coordinated release, not a same-branch edit.
- **ffmpeg is a subprocess**, not cgo/libav. `aioz-stream-core` shells out
  (`internal/utils/ffprobe/ffprobe.go:142`) with no cgo. CI `test:compile-check`
  cross-compiles `CGO_ENABLED=0 -tags purego` for linux/arm64, windows, freebsd,
  darwin -- a libav binding breaks that matrix.
- **Schema**: real tables. The spec cites `storage_configs` as its precedent but
  migration `000029` DELETED that table (per-contract config rejected in favour of
  static coord config + `contracts.placement_id`). Rationale does not transfer: a
  ladder is per-client arbitrary-cardinality data, unlike segment size.

## Two things the spec missed

1. **`paidActions` is a whitelist.** `coord/order/settlement.go:238` lists
   GET/PUT/GET_AUDIT/GET_REPAIR/PUT_REPAIR and `verifyOrder` rejects everything
   else ("limit action %s is not settleable", line 258). TRANSCODE must be added
   or every transcode settlement silently fails. DONE (Phase 3).
2. **`used_for_transcode` is DiskSpace field 12** (`model` holds 11).

## Status

- **Phase 1 (protos) DONE.** `TRANSCODE = 8` in PieceAction (last was 7);
  `TranscodeCapability transcode = 9` in CheckInRequest (last was 8);
  `used_for_transcode = 12` + `TranscodeCapability`/`HwaccelMode` in worker.proto;
  new `pkg/pb/coord/transcode/v1` + `pkg/pb/worker/transcode/v1`.
- **Phase 2 (coord config+session) DONE.** Migration `000055_transcode`
  (configs/renditions/sessions/disputes), `coord/transcode/` package,
  `coord/db/transcode_repo.go`, wired into BOTH `coord/peer.go` and `coord/api.go`,
  gated `--transcode.enabled=false`. 12 unit tests + 4 Postgres repo tests green.
- **Phase 3 (order limits) DONE 2026-09-10.** `NewSignerTranscode` +
  `Signer.SignTranscode` in `coord/order/signer.go`, `CreateTranscodeOrderLimits`
  + `TranscodeSegment`/`TranscodeAssignment` in new `coord/order/transcode.go`,
  `TRANSCODE` added to `paidActions` (`coord/order/settlement.go`). 7 tests in
  `coord/order/transcode_test.go`, all green; `go vet ./coord/...` clean;
  `CGO_ENABLED=0 -tags purego ./cmd/coord` builds.
- **Phases 4-8 NOT STARTED**: capability advertisement + selection (which is what
  wires the mint path into `GetTranscodeWorkers` -- the endpoint method does not
  exist yet), worker service, ffmpeg encode core, metering, testplanet e2e.

## Phase 3 decisions and findings

- **Spec risk #2 resolved: piece_id CARRIES the content hash verbatim.** No
  `TranscodeLimitMetadata` message. `vo.PieceID` is exactly 32 bytes = SHA-256,
  the Phase-1 proto already documents it (`SegmentSpec.content_hash` -> "carried
  in the limit's piece_id field"), and nothing downstream assumes a piece_id is
  derivable -- `usedserials` keys on (coordinator, serial), not piece_id.
- **`Signer.Sign` was refactored, not duplicated.** Limit construction moved into
  an unexported `signer.sign(ctx, workerID, addr, pieceID)`; `Sign` passes the
  DERIVED id (`rootPieceIDDeriver.Derive`), `SignTranscode` passes `RootPieceID`
  unchanged. One signing site still, as before.
- **One signer PER SEGMENT, not per batch.** That is what gives each segment its
  own serial AND its own uplink private key. A shared serial would make every
  submission after the first a replay.
- **Proto gap found and fixed: `TranscodeAssignment` had no private key.** Added
  `bytes piece_private_key = 5`. Without it the client cannot sign the order the
  worker settles, so the whole payment path was inert. `file.proto:199,394` is
  the precedent. Regenerated with `make proto`.
- Mint refuses a non-positive `ComputeUnits` and a nil/zero content hash. A
  zero-limit authorization is unsettleable, so minting one hands a worker work it
  can never be paid for. NOTE this collides with the known audio-only gap:
  `ComputeUnits` returns 0 for audio rungs, so an audio-only session now ERRORS
  at mint rather than minting a free budget. Still a pricing decision.
- The plan's "mint -> verify -> SettlementWithWindow -> replay rejected" walk is
  only PARTIALLY done: `usedserials` is worker-side in-memory state
  (`worker/piecestore/usedserials`), and there is no worker transcode service
  until Phase 5. `TestCreateTranscodeOrderLimits_SerialsAreSingleUse` runs the
  minted serials through the real `usedserials.Table`; the full window walk is
  Phase 8 e2e.
- `usedserials` imports `aioz-depin/internal/memory`, NOT `storj.io/common/memory`.

## Rung subsetting (added 2026-09-07, after Phase 2)

A profile is a SUPERSET ladder; `transcode_sessions.rendition_indexes` (JSONB)
picks the subset per video. One profile serves every video shape instead of a
profile per combination (subsets are combinatorial, and profiles freeze on first
use). Budget sums selected rungs only, so a 1080p master costs less than a 4K one.

**Selected rungs keep their PROFILE index** - never renumbered to 0..n, or
`Fetch(task, i)` means a different rung in different sessions. Duplicate index is
REFUSED, not deduplicated (hides a client bug while halving the expected bill).

Known gap, NOT fixed: `ComputeUnits` returns 0 for audio rungs (audio is a
whole-source rendition, so it has no pixel-seconds). An audio-only session
therefore mints a zero budget - audio compute is currently free. Needs a pricing
decision, not a code fix.

## Gotchas hit

- `errs/v2` `Tag` has `Errorf`/`Wrap` but **no `New`**, and the import is
  `github.com/zeebo/errs/v2` (v1 has no `Tag`).
- Coord protos use `go_package = "aioz-depin/pkg/pb"` -> generated Go `package pb`
  while living in `.../v1/` dirs, imported by path with an alias. Worker protos use
  the full path -> `package v1`. Match whichever side you are on.
- `buf lint` has ~106 PRE-EXISTING findings and is not run in CI (CI lint =
  golangci-lint + `make lint`). `buf breaking` is broken in-repo:
  "RPC_NO_DELETE_STREAMING is not a known rule".
- Domain type for a `transcode_configs` row is `Profile`, not `Config` -- the
  package's cfgstruct `Config` owns that name.
- GORM soft-delete scoping does NOT apply: the embedded generated message types
  `deleted_at` as `*time.Time`, not `gorm.DeletedAt`. Every query filters
  `deleted_at IS NULL` explicitly.
- Postgres truncates timestamps to microseconds. A mutating call that returns the
  in-memory `now` disagrees with a later read; `CloseSession` re-reads after the
  update. A test caught this.
- `depin-pgtest` is user `depin`, password `depin`, db `depin_test`, port 15445 --
  NOT user `postgres`.
- GORM `serializer:json` writes SQL **NULL** for a nil slice, which violates a
  NOT NULL column. Normalise nil -> empty in the repository, not at the caller.

## Verification commands

```
make proto && go build ./... && CGO_ENABLED=0 go build -tags purego ./cmd/coord
go test ./coord/transcode/ ./coord/order/ -count=1
DEPIN_TEST_POSTGRES="postgres://depin:depin@localhost:15445/depin_test?sslmode=disable" \
  go test ./coord/db/ -run TestTranscode -count=1
```
