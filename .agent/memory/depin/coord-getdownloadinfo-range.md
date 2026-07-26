---
type: decision
tags: [coord, getdownloadinfo, range, download, order-limits, bandwidth, erasure, encryption]
created: 2026-07-21
agent: main
---

**Coord `GetDownloadInfo`/`GetDownloadInfoByTicket` now support a plaintext byte range**
(Phase 1 of ranged downloads). Coord-only, stripe-exact accounting. go-sdk + edge are later
phases (intentionally untouched). Uncommitted (no-auto-commit). Builds + unit tests green.
Follows the finding that workers already serve piece byte offsets (see [[edge-outbound-bandwidth]]).

## What shipped
- **Proto** (`pkg/pb/coord/file/v1/file.proto`, regenerated depin's own file.pb.go via `buf generate`;
  go-sdk vendored copy deliberately NOT resynced - that's Phase 2):
  - new `message PlainRange { int64 offset=1; int64 length=2; }`.
  - nullable `PlainRange range` on `GetDownloadInfoRequest` + `GetDownloadInfoByTicketRequest`
    (nil = whole file, backward compatible; ticket still whole-file, range is a read param).
  - `int64 plain_offset` on `DownloadSegment` (segment's object-space plaintext start; needed
    because ranged manifests omit non-covering segments so the client can't accumulate it).
  - `int64 piece_offset` on `RemoteSegment` (stripe-aligned per-piece byte offset the order limits
    are scoped to; per-piece length is the order limit's existing `Limit`).
- **`coord/file/range.go`** (new): `pieceByteRange(scheme, params, encryptedSize, segStart, segEnd)`
  maps a segment-relative plaintext range -> encrypted blocks -> stripe/piece window. Reuses
  `pkg/encryption` (`CalcEncompassingBlocks`, `NewEncrypter` for InBlockSize/OutBlockSize with
  `DefaultBlockSize`=256; AES-GCM plaintext block=240, encrypted=256) + `RedundancyScheme.StripeSize/
  PieceSize`. Handles RS (stripe-aligned) and CLONE (no erasure -> encrypted-block window directly).
  Rounds OUTWARD to block+stripe boundaries (client trims).
- **`coord/file/endpoint.go`**: `buildManifest(ctx, fileUUID, rng)` normalizes the range
  (`normalizeRange`: nil=whole file; length<=0 or past-EOF clamps to EOF; offset OOB -> OutOfRange
  err the edge maps to 416), iterates segments accumulating plainOffset, selects covering segments,
  computes a `plainWindow{start,end}` per covering segment, and sets `ds.PlainOffset`.
  `buildDownloadSegment(..., win, ...)`: win==nil keeps the exact prior whole-segment behavior;
  win!=nil scopes the order-limit amount (pass `pr.pieceLen` as the `pieceSize` arg to the existing
  `orders.CreateGetOrderLimits` - NO order-service change), sets `RemoteSegment.PieceOffset`, and
  bills `usage.egress += pr.encRangeBytes` + `allocated += pieceLen` per worker (scoped).

## Key facts / gotchas
- **Whole-file path is byte-identical**: rng==nil never calls pieceByteRange; PlainOffset/PieceOffset
  stay 0 (proto3 zero not serialized). Existing tests unchanged & green.
- **Two +4 length prefixes**: one in the encryption layer (inside EncryptedSize via CalcEncryptedSize)
  and one in the RS/eestream layer (PieceSize adds +4 before stripe pad). The ciphertext sits at the
  FRONT of the RS-encoded stream, so encrypted offset O maps directly to encoded offset O; only the
  last stripe/total is affected. totalStripes = PieceSize(encSize)/ShareSize.
- **`Segment.RedundancyScheme` is `file.RedundancyScheme` = `sharedpb.RedundancyScheme`** (has
  StripeSize/PieceSize, `.Algorithm` is `sharedpb.Algorithm`), NOT the other `RedundancyScheme1`
  (redundancy_scheme.go, byte Algorithm). `Segment.EncryptionParameters` = `common.EncryptionParameters`
  (field `CipherSuite` is `sharedpb.CipherSuite`; build one with `{CipherSuite: sharedpb.CipherSuite_ENC_AESGCM}`).
- **Worker does NOT enforce order-limit amount on download** (clamps to piece bounds only; the
  `.Limit`/`.Amount` checks in worker/piecestore/endpoint.go are the UPLOAD path). So scoped limits
  are for accounting; binding/enforcing piece_offset worker-side is deferred hardening.
- **Deliberate non-goal**: piece_offset is NOT signed into the order limit; a client could read more
  of a piece than authorized (within the piece). Fine for now (settlement records actual bytes,
  allocated>=settled holds).
- **testplanet e2e still blocked** by the pre-existing `CreateContractWithPlacement` go-sdk drift
  (unrelated). Covered by coord unit tests: `coord/file/range_test.go` (pieceByteRange RS+CLONE with
  hand-verified magic numbers + invariants; normalizeRange) + a CLONE ranged assertion appended to
  `endpoint_metrics_test.go`'s TestBuildDownloadSegment_KindTagging. All green, vet+gofmt clean.

## Follow-ons (not done)
- Phase 2 (go-sdk): lazy piece `ranger.Ranger` over `GetPiece(offset,length)` -> vendored
  `eestream.decodedRanger.Range` + encryption transform/unpad rangers; consume plain_offset/piece_offset;
  resolve order SerialNumber reuse for multiple ranged GETs. Then resync go-sdk's vendored file.pb.go.
- Phase 3 (edge): true 206/Content-Range/Content-Length + suffix/open ranges.
- Related: [[edge-outbound-bandwidth]].
