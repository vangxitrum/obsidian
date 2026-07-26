---
type: decision
tags: [depin, coord, file, delete]
created: 2026-07-08
agent: main
---

Implemented Phase 1 (metadata delete) of the delete-file feature per plan `projects/depin/plans/2026-07-08-delete-file.md`, on isolated worktree `/home/tuan/.treehouse/depin-b971d9/1/delete-file-wt`, branch `feat/delete-file`. Uncommitted — awaiting user review.

Changes:
- `pkg/pb/coord/file/v1/file.proto`: added `rpc DeleteFile` + `DeleteFileRequest`/`DeleteFileResponse` (buf-generated).
- `coord/file/store.go`: `DeleteFileResult{Removed *File, DeletedSegmentCount int64}` + `Store.DeleteFile`.
- `coord/db/file_repo.go`: `fileRepository.DeleteFile` — single-round-trip raw SQL CTE (`WITH deleted_file AS (DELETE FROM files ... RETURNING *), deleted_segments AS (DELETE FROM segments ... RETURNING file_id) SELECT ...`), mirrors storj metabase `DeleteObjectExactVersion`. Hard delete, not gorm soft-delete (`deleted_at` avoided — would skip FK cascade, leave orphan segments visible to future GC keep-set scan).
- `coord/file/service.go`: `Service.Delete(ctx, fileUUID, caller)` reuses existing `AuthorizeFileOwner`, wraps store call in `WithTx`. Idempotent (missing file = empty result via store, but AuthorizeFileOwner still returns NotFound before reaching store — so a *second* delete call on an already-gone file returns NotFound too, not silently OK — this diverges slightly from pure idempotency but matches the plan's reliance on AuthorizeFileOwner).
- `coord/file/endpoint.go`: `Endpoint.DeleteFile` gRPC handler, same pattern as `GetDownloadInfo` (mTLS peer identity + `vo.GrpcError` mapping). No `coord/peer.go` change needed — `filepb.RegisterFileServiceServer` already wires all `FileServiceServer` methods.
- Added `TestServiceDelete` in `coord/file/service_test.go` (owner match / wrong caller Forbidden / missing file NotFound), plus `fakeContractStore` test double. All green.

**Why:** mirrors Storj delete semantics exactly (metadata-only; pieces reclaimed lazily by GC) per user's stated design decisions in the plan.

**How to apply:** Phase 2 (GC bloom-filter observer + sender + worker Retain RPC) is a separate, larger follow-up — machinery exists but is dormant (see plan section "Phase 2"). DB-backed store test and full e2e cluster test (uplink-sdk) still need to be run against real Postgres/cluster before merge. SDK client wrapper intentionally out of scope.

Related: [[hub-overview]]
