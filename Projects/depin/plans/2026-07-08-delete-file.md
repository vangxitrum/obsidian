# Delete File — Implementation Plan

## Context

Depin lets users upload files (SDK → coordinator `FileService` → workers) but there is **no way to delete one**. `coord/file/service.go:29` even notes: "a committed file stays in the network until deleted" — the delete path was never built. Users need to remove uploaded files (reclaim contract quota, honor deletion requests).

We mirror Storj (`../storj`). Per `storj/delete-object.md`, Storj delete is **metadata-only**: the satellite drops the object + segment rows and returns no piece list; storage nodes are never told. Pieces become orphans reclaimed **lazily** by garbage collection — a rangedloop observer builds a per-node bloom filter of pieces to KEEP, a sender ships it, the node trashes everything not in the filter, and trash is purged after a 7-day grace.

**Intended outcome:** a `DeleteFile` capability, keyed by durable file UUID, mirroring Storj's delete request path exactly. This plan delivers the coordinator-side **metadata delete** (Phase 1). Actual disk reclamation via GC is **Phase 2** (separate, larger; the machinery is present but dormant).

**Execution:** implement on an isolated **git worktree**, not the current workspace.

**Scope note:** the SDK client method is intentionally OUT of scope (per user). Phase 1 delivers the proto RPC + coordinator server side only; a caller invokes it via the generated `FileServiceClient` stub directly.

### Design decisions (defaults chosen; user was away — revisit if wrong)
- **Metadata-only delete**, matching Storj. NOT direct piece-wipe (Storj abandoned that path).
- **Hard delete** file + segment rows in a transaction (segment cascade drops inline data). Storj-aligned; keeps the rangedloop keep-set correct for Phase-2 GC; worker trash 7d is the safety net. We do NOT use the `deleted_at` gorm soft-delete column (it would skip the FK cascade and leave orphan segment rows in the live scan, breaking keep-set GC).

## Phase 1 — Metadata delete (the feature)

Data path mirrors `DownloadFile` / `CommitFile`: `FileService` RPC → `Endpoint` handler → `Service` method (reusing `AuthorizeFileOwner`) → `Store` tx.

### Storj layer mapping (keep names/shapes aligned; Object→File as depin already does)
| depin layer | storj counterpart (source) |
|---|---|
| RPC `DeleteFile` | metainfo `BeginDeleteObject` (`endpoint_object_delete.go:29`) — depin drops the `Begin` prefix for file-level ops, as `CommitFile` does for `CommitObject` |
| `Endpoint.DeleteFile` handler | `Endpoint.DeleteCommittedObject` (`endpoint_object_delete.go:182`) |
| `Service.Delete` | thin, delegates to store (storj does this in the endpoint) |
| `Store.DeleteFile` + `DeleteFileResult` | metabase `DeleteObjectExactVersion` + `DeleteObjectResult{Removed, DeletedSegmentCount}` (`delete.go:70,80`) |
| CTE deleting `files` then `segments` | `delete.go:120-143` — `DELETE FROM objects ... ; DELETE FROM segments WHERE stream_id IN (deleted)` |

### 1. Proto — add the RPC
File: `pkg/pb/coord/file/v1/file.proto`
- Add to `service FileService` (after `CommitFile`, `:42`):
  ```
  rpc DeleteFile(DeleteFileRequest) returns (DeleteFileResponse);
  ```
- Add messages (mirror `GetDownloadInfoRequest` `:297` — keyed by durable UUID):
  ```
  message DeleteFileRequest {
    bytes file_id = 1 [(gogoproto.customtype)="aioz-depin/internal/vo.UUID",(gogoproto.nullable)=false];
  }
  message DeleteFileResponse {}
  ```
- Regenerate: `buf generate` (driven by `buf.gen.yaml` / `buf.yaml` at repo root). Produces updated `file.pb.go` + `file_grpc.pb.go`.

### 2. Store — add hard-delete method (mirrors metabase `DeleteObjectExactVersion`)
Files: `coord/file/store.go` (interface + result struct) + `coord/db/file_repo.go` + `coord/db/file_store.go` (gorm impl)
- Add a storj-shaped result struct in `coord/file` (mirror `DeleteObjectResult`, `delete.go:70`):
  ```go
  // DeleteFileResult mirrors storj metabase DeleteObjectResult.
  type DeleteFileResult struct {
      Removed             *File // nil if nothing matched
      DeletedSegmentCount int64
  }
  ```
- Add to the `Store` interface (next to `DeleteExpiredPendingFiles`, `store.go:36`):
  ```go
  // DeleteFile hard-deletes the file row and its segments, mirroring storj
  // metabase DeleteObjectExactVersion. Returns the removed file + segment count.
  // A missing file yields an empty result (idempotent), not an error.
  DeleteFile(ctx context.Context, id vo.UUID) (DeleteFileResult, error)
  ```
- Impl in `coord/db/file_repo.go`: run the same explicit two-statement delete storj uses (`delete.go:120-143`), as a raw SQL CTE via gorm `Raw/Exec` inside the caller's tx:
  `WITH deleted_file AS (DELETE FROM files WHERE id = ? RETURNING ...), deleted_segments AS (DELETE FROM segments WHERE file_id IN (SELECT id FROM deleted_file) RETURNING file_id) SELECT ..., (SELECT COUNT(*) FROM deleted_segments)`.
  This mirrors storj's CTE and returns the segment count directly (don't rely on gorm's soft-delete or implicit cascade — be explicit, like storj). Inline segment data lives in the segment row so it goes with the segment delete.
- Delegate through `coord/db/file_store.go` like the other `Store` methods.

### 3. Service — add `Delete`, reuse the owner-auth path (mirrors `DeleteCommittedObject`)
File: `coord/file/service.go`
- Add method next to `CommitFile` (`:249`):
  ```go
  // Delete removes a file and all its segments (metadata-only, mirrors storj
  // Endpoint.DeleteCommittedObject → metabase DeleteObjectExactVersion).
  // Owner-authorized via mTLS identity + DB owner lookup. Pieces on workers are
  // reclaimed later by GC (Phase 2). Idempotent.
  func (s *Service) Delete(ctx context.Context, fileUUID, caller vo.UUID) (DeleteFileResult, error) {
      defer mon.Task()(&ctx)(&err)
      if _, err := s.AuthorizeFileOwner(ctx, fileUUID, caller); err != nil {
          return DeleteFileResult{}, err // NotFound / Forbidden already wrapped
      }
      var res DeleteFileResult
      err := s.store.WithTx(ctx, func(tx Store) (err error) {
          res, err = tx.DeleteFile(ctx, fileUUID)
          return FileError.Wrap(err)
      })
      return res, err
  }
  ```
- Reuses **`AuthorizeFileOwner` (`service.go:111`)** — no new auth logic. Works for READY and PENDING files (delete does not gate on status, unlike download).

### 4. Endpoint — add the gRPC handler (mirrors `BeginDeleteObject`)
File: `coord/file/endpoint.go` (storj isolates this in `endpoint_object_delete.go`; depin keeps one `endpoint.go`, so add here next to the download handlers `:587`+)
- Add `DeleteFile(ctx, *pb.DeleteFileRequest) (*pb.DeleteFileResponse, error)`. Pattern = `GetDownloadInfo`: extract caller UUID from the mTLS peer identity the same way existing durable-op handlers do, call `res, err := s.service.Delete(ctx, req.FileId, caller)`, map errors (`vo.NotFound`→codes.NotFound, `vo.Forbidden`→codes.PermissionDenied) as the sibling handlers do, optionally log `res.DeletedSegmentCount` (storj meters `object_delete`/`segment_delete`), return empty response.
- No registration change needed — `filepb.RegisterFileServiceServer` (`coord/peer.go:392`) picks up the new method.

### 5. SDK — OUT of scope
No `uplink-sdk` method in this plan (per user). The RPC is reachable via the generated `FileServiceClient.DeleteFile` stub; the SDK wrapper can be added later.

## Phase 2 — Piece reclamation via GC (follow-up, larger; document as TODO)

Not required to ship the feature; without it deleted pieces orphan on worker disks (Storj accepts this — reclamation is lazy). The machinery is **present but dormant** — Phase 2 is mostly wiring + porting the coord producer:

1. **Coord bloom-filter observer** (MISSING) — port `storj/satellite/gc/bloomfilter/observer.go` into `coord/gc/bloomfilter/`, register it on the rangedloop alongside audit/repair observers (`coord/peer.go:503-532`). It scans live segments, derives per-worker piece IDs via the existing `internal/vo/piece_id.go:170` deriver, builds per-worker keep-set bloom filters.
2. **Coord gc/sender** (MISSING) — port `storj/satellite/gc/sender/`; dials workers, sends `Retain`.
3. **Worker `Retain` gRPC** (MISSING) — add `Retain`/`RestoreTrash` to `PieceStoreService` (`pkg/pb/worker/piece/v1/piece.proto`; `RetainRequest`/`RetainResponse` messages already exist at `piece.pb.go:1302`), handler in `worker/piecestore/endpoint.go`.
4. **Wire the dormant queue** — `worker/peer.go:520-521` passes `nil` for `retainQueue`/`restoreTrash`; feed the already-built `worker/retain` `Service` + `RequestStore`. Bloom read-path and `TrashChore` (`worker/retain/chore.go`, trash + 7d) are already constructed (`worker/peer.go:405-423`).

## Files touched (Phase 1)
- `pkg/pb/coord/file/v1/file.proto` (+ regenerated `file.pb.go`, `file_grpc.pb.go`)
- `coord/file/store.go`, `coord/db/file_repo.go`, `coord/db/file_store.go`
- `coord/file/service.go`
- `coord/file/endpoint.go`

## Verification (Phase 1, end-to-end)
1. **Build:** `go build ./...` after buf regen.
2. **Unit — service:** table test `Service.Delete` with a fake `Store`: owner match → `DeleteFile` called; wrong caller → `vo.Forbidden`, no delete; missing file → `vo.NotFound`. Mirror existing `coord/file` service tests.
3. **DB — store (needs Postgres):** insert a file + segments (one remote, one inline) → `DeleteFile` → assert both `files` and `segments` rows gone (cascade), inline data gone. Follow the Postgres-backed repo test pattern already in `coord/db`.
4. **E2E (real cluster, via gRPC stub):** in `uplink-sdk/test/integration_test.go` (or a coord-side endpoint test), upload a file, confirm download works, then call the generated `FileServiceClient.DeleteFile` stub directly (no SDK wrapper): download again → expect NotFound; call `DeleteFile` a second time → idempotent (NotFound, no crash); call `DeleteFile` as a different mTLS identity → PermissionDenied. Drives the real handler over the wire — closest reproduction of the delete request path.
