# Plan: user-settable file metadata (coordinator slice)

## Context

Today a stored file in depin carries no user-settable metadata. The `files` table
has only `id, contract_id, name, size, status, created_at, deleted_at`, the
download manifest carries only `file_id, file_size, segments[]`, and the edge
server hardcodes `Content-Type: application/octet-stream`
(`edgeserver/handler.go:29`). There is no way for a client to say "this object is
`image/png`" or attach arbitrary tags, and therefore no way for the edge server to
emit a correct `Content-Type` or custom response headers.

Goal: add S3-style file metadata — a `content-type`, the other settable S3 system
headers, and arbitrary user key/value pairs — set once at upload (immutable), so a
later edge iteration can turn them into HTTP response headers.

Design decisions already made with the user:
- **Full S3-style**: `content_type` + settable system headers + arbitrary user pairs.
- **Plaintext in the coordinator DB** (Approach A). depin already stores filenames
  in plaintext, so it is not zero-knowledge; storing metadata plaintext is
  consistent and avoids the crypto plumbing Storj uses. Coordinator enforces limits
  authoritatively and can index on content-type later.
- **Limit = total bytes, S3-exact**: `Σ(len(key)+len(value))` over user pairs ≤ 2048.
  No pair-count cap. System headers do not count toward this budget (matches S3).
- **Immutable**: set at `CreateFile`, never mutated. No update RPC.
- **Scope for THIS iteration = coordinator backend only.** Per the user: skip ALL
  `uplink-sdk` changes. That also defers the edge-server handler change (the edge
  reads through the SDK download path, which we are not touching). We land the DB
  columns, the wire contract on `CreateFileRequest`/`DownloadManifest`, coordinator
  persistence, and validation. Nothing populates metadata in production yet; it is
  fully backward compatible (unset proto fields → empty → validation passes → edge
  keeps its current hardcoded behavior) and is verified via direct coordinator gRPC.

## Changes

### 1. Migration — `coord/db/migrations/000031_file_metadata.{up,down}.sql`

Follows the established add-column + JSONB pattern (`000020_worker_egress_nets`,
`annotations jsonb` in `000001`).

`up`:
```sql
ALTER TABLE files
    ADD COLUMN IF NOT EXISTS content_type   TEXT,
    ADD COLUMN IF NOT EXISTS system_headers JSONB,
    ADD COLUMN IF NOT EXISTS user_metadata  JSONB;
```
`down`: `ALTER TABLE files DROP COLUMN IF EXISTS content_type, DROP COLUMN IF EXISTS system_headers, DROP COLUMN IF EXISTS user_metadata;`

No test-schema mirror needed — `coorddbtest` runs the real migrations against a
throwaway schema (`coord/db/coorddbtest/schema.go`).

### 2. Persistence model — `coord/file/file.go`

Add three fields to the `File` wrapper struct (NOT to `pb.File`, keeping the wire
`File` message untouched), reusing the proven `serializer:json` + JSONB pattern
from `coord/placement/placement.go:59` and `coord/contact/worker.go:48`:
```go
type File struct {
    pb.File
    Contract      *contractpb.Contract `gorm:"foreignKey:ContractId"`
    ContentType   string               `gorm:"column:content_type"`
    SystemHeaders map[string]string    `gorm:"column:system_headers;serializer:json"`
    UserMetadata  map[string]string    `gorm:"column:user_metadata;serializer:json"`
}
```
gorm `Create`/`First` (`coord/db/file_repo.go:18,34`) pick these up automatically —
no repo changes required.

### 3. Validation + metadata type — new `coord/file/metadata.go`

- `type FileMetadata struct { ContentType string; SystemHeaders, UserMetadata map[string]string }`.
- `settableSystemHeaders` = set of lowercase keys `{content-disposition, cache-control,
  content-encoding, content-language, expires}` (content-type is its own field, not
  in this set).
- `const MaxUserMetadataBytes = 2048`.
- `func (m *FileMetadata) Normalize()` — lowercase all keys (S3 behavior) for stable
  header naming.
- `func (m FileMetadata) Validate() error` returning `vo.Validate`-tagged errors
  (so `vo.GrpcError` → `codes.InvalidArgument`, per `internal/vo/errors.go:67`):
  1. Every `system_headers` key ∈ `settableSystemHeaders`, else reject (unknown
     system header).
  2. `Σ(len(k)+len(v))` over `user_metadata` ≤ `MaxUserMetadataBytes`, else reject.
  3. **Security — HTTP header injection:** reject any key or value (content-type,
     system, user) containing CR, LF, or control chars (`< 0x20` or `0x7f`). These
     values will later flow into edge HTTP response headers; block CRLF injection at
     ingest.
- Unit-tested in isolation (see Testing).

### 4. Wire-through — proto + service + endpoint

- **`pkg/pb/coord/file/v1/file.proto`** — add to `CreateFileRequest` (next free tags,
  e.g. 8/9/10) and `DownloadManifest` (next free tags 4/5/6):
  ```proto
  string              content_type   = N;
  map<string, string> system_headers = N;
  map<string, string> user_metadata  = N;
  ```
  No gorm `moretags` on these — persistence lives on the wrapper struct, these are
  pure wire fields. Regenerate with `buf generate` (updates `file.pb.go`).
- **`coord/file/service.go` `Service.Create`** — add a `meta FileMetadata` param.
  At the top: `meta.Normalize()` then `if err := meta.Validate(); err != nil { return ... FileError.Wrap(err) }`.
  Set `newFile.ContentType/SystemHeaders/UserMetadata` before `store.Create`.
- **`coord/file/endpoint.go` `CreateFile` (:86)** — build `FileMetadata` from
  `req.ContentType/SystemHeaders/UserMetadata`, pass to `service.Create`. Existing
  `vo.GrpcError(err)` at :104 already maps validation failures to `InvalidArgument`.
- **`coord/file/endpoint.go` `buildManifest` (:683)** — after `LoadDownloadData`
  returns `file`, copy `file.ContentType/SystemHeaders/UserMetadata` onto the
  `DownloadManifest`. Flows out through both `GetDownloadInfo` and
  `GetDownloadInfoByTicket` unchanged.

`LoadDownloadData` already returns the full `*File`, so no service-layer read change.

## Out of scope (explicit follow-ups)

These consume the new `DownloadManifest` fields and are deferred with the SDK:
- `uplink-sdk/upload.go` upload params + `cmd/uplink` `--content-type` / `--meta` flags
  (the client "set" path).
- `uplink-sdk/download.go` surfacing manifest metadata to callers.
- `edgeserver/handler.go` replacing the hardcoded `Content-Type` with
  `manifest.content_type` (fallback `application/octet-stream`), mapping
  `system_headers` → canonical HTTP headers, and `user_metadata` → `x-aioz-meta-<k>`
  (AIOZ-branded prefix, the depin analog of S3's `x-amz-meta-`).

Also out of scope: mutable/update RPC, encrypted metadata, ETag/conditional GET.

## Verification

1. `buf generate` — regenerate proto; confirm new fields on `CreateFileRequest` and
   `DownloadManifest` in `file.pb.go`.
2. `go build ./...` and `go vet ./coord/...`.
3. `go test ./coord/file/...`:
   - New `coord/file/metadata_test.go`: `Validate` rejects unknown system header,
     rejects `user_metadata` at 2049 bytes and accepts at 2048 (boundary), rejects
     CR/LF/control chars in key and value, and `Normalize` lowercases keys.
   - Extend `coord/file/service_test.go`: `Create` with valid metadata persists it;
     `Create` with oversized/invalid metadata returns a `vo.Validate`/InvalidArgument
     error and writes nothing.
4. **Coordinator round-trip (closest to end-to-end for this slice; needs Postgres via
   `coorddbtest`)**: call `CreateFile` with `content_type` + system + user maps, then
   `GetDownloadInfo` / `GetDownloadInfoByTicket`, and assert the returned
   `DownloadManifest` carries `content_type`, `system_headers`, and `user_metadata`
   back byte-for-byte. This exercises the whole coordinator path the future edge work
   will rely on. (True uplink→edge e2e is deferred with the SDK/edge changes.)
5. Backward-compat check: an existing-style `CreateFile` with no metadata still
   succeeds and yields empty metadata fields (proto3 zero values).

After edits, re-verify any `for`/loop bodies touched by the formatter (known
auto-formatter hazard). Do not commit unless the user explicitly approves.
