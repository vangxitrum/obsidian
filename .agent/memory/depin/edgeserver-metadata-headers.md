---
type: decision
tags: [edgeserver, go-sdk, metadata, http-headers, testplanet, infectious, gitignore]
created: 2026-07-12
agent: main
---

**Edgeserver serves file metadata as HTTP response headers** (download side only).
Branch: work on `develop` tree, uncommitted (no-auto-commit rule). Complements
[[edgeserver-go-sdk-migration]].

## What shipped
- **edgeserver/handler.go**: `handleDownload` now registers a header sink. System
  headers (the 6 coord-allow-listed: content-type, cache-control,
  content-disposition, content-encoding, content-language, expires) are set
  directly (`w.Header().Set(lowercaseKey, v)`; Go canonicalizes). Default
  `Content-Type: application/octet-stream` only when absent. User metadata →
  `X-User-Meta-<key>` (validated via `httpguts.ValidHeaderFieldName`, skip
  illegal). Ticket now passed as a raw **string** (SDK decodes) — no local
  base64 decode. Content-Length intentionally NOT set (streamed).
- **edgeserver/server.go**: added `NewFromClient(client, logger)` + `Handler()
  http.Handler` (embeddable + testable; shared `newServer` builds the mux).
- **go-sdk/download.go**: `DownloadByTicket(ctx, ticket, w, opts...)` gained
  `WithHeaderSink(func(system, user map[string]string))`, fired after ticket
  resolve, before first body byte, with `manifest.GetSystemHeaders()/GetUserMetadata()`.
- **go-sdk proto resync**: coord already stamped `system_headers`(4)/`user_metadata`(5)
  onto `DownloadManifest` (coord/file/endpoint.go buildManifest), but go-sdk's
  vendored proto was stale and dropped them. Fix = copy depin's
  `pkg/pb/coord/file/v1/file.pb.go` into go-sdk, apply `hack/sync-from-depin.sh`
  transforms manually (that script early-exits now — guards on the deleted
  `uplink-sdk/`): sed `aioz-depin/internal/`→`aioz-depin/go-sdk/internal/` and
  `aioz-depin/pkg/`→`aioz-depin/go-sdk/pkg/`, then `sed -i '/proto\.RegisterEnum(/d'`.

## Gotchas hit (durable)
1. **go-sdk pkg/infectious missing `addmul_amd64.s`** (the allowlist-.gitignore
   drops-.s gotcha, see [[depin-gitignore-allowlist-gotcha]]). No go-sdk module
   version ships it. `addmul_noasm.go` is `//go:build !amd64 || purego`, so on
   amd64 WITHOUT `-tags purego` the asm path is required → depin `go build ./...`
   fails `missing function body` on addmulSSSE3/addmulAVX2. Fix = copy depin's
   identical `pkg/infectious/addmul_amd64.s` into go-sdk. **When committing go-sdk,
   `git add -f` the .s or it silently drops again and breaks fresh clones/CI.**
2. **Re-pin to go-sdk HEAD drags in unrelated commits.** Pinned was `bb4f12b`;
   sibling HEAD `c018645` is +2 (2fdadc4 piece-key auto-gen, c018645 ticket→string).
   2fdadc4 changed `piece_key.json` format to `{"private_key": "<hex-64>"}` (was
   `{"key": ...}`) → testplanet `writeUplinkIdentityDir` (internal/testplanet/uplink.go)
   failed `load piece key: invalid private key size`. Fixed the JSON field name.
3. **RegisterMapType/RegisterType only LOG on duplicate; only RegisterEnum panics**
   (verified in regen-network/protobuf proto/properties.go). So the new
   `RegisterMapType(...SystemHeadersEntry)` lines are safe to keep in the dual-link
   (testplanet links both go-sdk pb + depin pb); the sync only drops RegisterEnum.
   The `proto: duplicate proto type registered` log at test start is expected/benign.

## Verify
`internal/testplanet/edgeserver_metadata_test.go` — e2e via `httptest` against the
real `edgeserver.Handler()`: upload, stamp metadata on the coord file row
(`coord.DB.GetDB().Model(&file.File{}).Where("id=?",fileID).Updates(&file.File{...})`
— serializer:json runs on struct Updates), mint ticket, GET, assert
Content-Type/Cache-Control/X-User-Meta-Foo + body + default-content-type case.
Run: `DEPIN_TEST_POSTGRES="postgresql://admin:admin123@localhost:5445/hub?sslmode=disable"
go test ./internal/testplanet/ -run TestEdgeserverMetadataHeaders`. PASSED.

## Not mine / pre-existing
- go-sdk `internal/vo` `TestPieceIDScanNullAndEmpty` fails (from go-sdk initial
  commit; unrelated).
- depin `uplink/application/service/upload_part_review_test.go` stale vs current
  MakeInlineSegmentRequest proto (test-only; doesn't block `go build ./...`).

## Finalization (was pending, done 2026-07-22)
go.mod's `replace aioz-depin/go-sdk => ../go-sdk` local iteration pin was resolved
in [[edge-download-timeout-fix]]: go-sdk committed/pushed to `origin/main`, depin
re-pinned to `replace aioz-depin/go-sdk => gitlab.internal/aioz-depin/go-sdk
v0.0.0-20260722153257-a9739e31ca2a` (computed via a scratch-module `go get`, since
the hostless require name can't resolve pseudo-versions directly), `go mod tidy`.
Recipe used, repeatable for future go-sdk bumps: build an isolated scratch Go
module elsewhere, `go get gitlab.internal/aioz-depin/go-sdk@<commit>` there to let
Go compute+print the real pseudo-version (fails with a "module declares its path
as X but was required as Y" error - that's expected/fine, the version was already
resolved before that error), then hand-apply that exact version string to depin's
real `require`+`replace` lines and run `go mod tidy` there.
