---
type: decision
tags: [go-sdk, ranged-download, download, eestream, encryption, testplanet, range]
created: 2026-07-21
agent: main
---

**Phase 2 of ranged downloads: go-sdk consumes coord's range-scoped manifest.** Builds on
[[coord-getdownloadinfo-range]] (Phase 1). Uncommitted (no-auto-commit). go-sdk unit tests +
a real testplanet e2e both green; depin `go build ./...` now FULLY clean.

## What shipped (go-sdk at /home/tuan/work/depin-workspace/go-sdk, symlinked ../go-sdk)
- **Proto resync**: manually copied depin's `pkg/pb/coord/file/v1/file.pb.go` into go-sdk + applied
  the import seds (`aioz-depin/internal/`->`aioz-depin/go-sdk/internal/`, `aioz-depin/pkg/`->
  `aioz-depin/go-sdk/pkg/`) + `sed '/proto\.RegisterEnum(/d'`. The full `hack/sync-from-depin.sh`
  is UNUSABLE (wipes all of go-sdk, requires the deleted `uplink-sdk/`); targeted file copy only.
- **`download.go`**: `DownloadByTicketRange`/`DownloadFileRange` (set `req.Range = &PlainRange{...}`,
  fire header sink, compute effective `[gStart,gEnd)` via `effectiveRange`), and
  `DownloadManifestRange` (per covering segment: intersect `[gStart,gEnd)` with
  `[PlainOffset, PlainOffset+PlainSize)`, call `segmentdownload.OpenRange`, concat).
- **`internal/segmentdownload/segment.go`**: `OpenRange` + `newSegmentDecrypter` (factored from
  `decryptStream`) + `decryptRangeTail`. RS path: `Fetch(pieceOffset,pieceLen)` -> `DecodeReaders2`
  over the scoped stripe window (expectedSize = numStripes*StripeSize) -> discard `encStart - S`
  (S = pieceOffset/ShareSize*StripeSize) -> `TransformReader(decrypter, firstBlock)` -> trim head +
  LimitReadCloser. CLONE: pieceOffset already block-aligned, no RS, decrypt from firstBlock. Inline:
  decrypt whole blob + slice.
- **`internal/piecedownload/manager.go`**: `Fetch` gains a `pieceOffset int64` param (whole-segment
  callers pass 0).
- **depin `internal/testplanet`**: drift fix (`CreateContractWithPlacement(...,nil)` - go-sdk added a
  `tags map[string]string` param); `Uplink.DownloadRange` helper; `TestRangedDownload` e2e.

## Key decode facts (the math that makes it correct)
- The SDK derives `firstBlock = segWantStart/plainBlock` (AES-GCM 240) itself; `encStart =
  firstBlock*encBlock` (256). Coord's `piece_offset` gives the stripe boundary `S <= encStart`.
  Discarding `encStart - S` bytes reaches the block boundary - **no share-size/block alignment
  assumption needed** (plain byte discard).
- Decrypt mid-stream: `TransformReader(enc, decrypter, firstBlock)` - the decrypter (built with
  startingNonce = baseNonce+segNum) derives each block's nonce as startingNonce+blockNum, so passing
  firstBlock is correct. `decryptStream` still passes 0 (unchanged whole-segment path).
- Order `SerialNumber` reuse is a non-issue: each ranged download resolves a fresh manifest -> fresh
  serials, each piece fetched once.

## Gotchas (durable)
- **Formatter mangles go-sdk imports**: the depin PostToolUse goimports hook, run on a go-sdk file,
  rewrites `aioz-depin/go-sdk/pkg/common` -> `aioz-depin/pkg/common` (resolves to depin's module).
  After editing any go-sdk file that imports `common`, re-check + fix the import. Caught by build.
- **LSP is stale for the symlinked go-sdk**: diagnostics show new go-sdk methods/proto fields as
  "undefined" (module cache), but `go build`/`go test` (which read the ../go-sdk symlink) are correct.
  Trust the CLI, not the LSP, for cross-module changes here.
- Test fixture: RS shares are produced by `PadReader(enc, strategy.StripeSize())` THEN
  `eestream.EncodeReader2` (the existing `segment_test.go` pattern); all TotalCount tee readers must be
  read concurrently (`readAllConcurrent`).

## Tests
- go-sdk `internal/segmentdownload/range_test.go`: in-memory round-trip, RS + CLONE + inline, ranges
  = block-aligned/mid-block/spanning-stripes/to-EOF/last-byte; the coord `pieceByteRange` math is
  replicated as `scopeRS`/`calcBlocks` to feed OpenRange coord-shaped inputs; ground truth is
  `plaintext[start:end]`.
- e2e `internal/testplanet/range_download_test.go` `TestRangedDownload`: upload 64 KiB remote segment,
  `DownloadRange` 7 ranges (incl. whole-via-range + to-EOF) through real coord+workers, assert slices.
  Run: `DEPIN_TEST_POSTGRES="postgresql://admin:admin123@localhost:5445/hub?sslmode=disable" go test
  ./internal/testplanet/ -run TestRangedDownload`. PASSED.

## Follow-on (not done)
Phase 3 (edge): swap the edge's closed-range stream-and-trim for `DownloadByTicketRange` behind real
`206`/`Content-Range: start-end/total`/`Content-Length` + suffix/open ranges. Also: finalize/pin
go-sdk (commit + pseudo-version) instead of the local `../go-sdk` symlink. Related:
[[coord-getdownloadinfo-range]], [[edge-outbound-bandwidth]].
