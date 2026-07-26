---
type: decision
tags: [edgeserver, bandwidth, egress, http-cache, etag, gzip, range, cdn, go-sdk]
created: 2026-07-19
agent: main
---

**Edge server outbound-bandwidth reduction, edge-only (no go-sdk changes).**
Implemented in `edgeserver/` on the depin worktree; go-sdk deliberately left
untouched per user directive. Complements [[edgeserver-metadata-headers]] and
[[edgeserver-go-sdk-migration]].

## What shipped (all in `edgeserver/`)
- **Canonical route** `GET /download/{fileId}` next to legacy `GET /download?ticket=`
  (`server.go`). Object id is a stable CDN cache key; ticket carried out-of-key
  (query or `Authorization: Bearer`). `handler.go: handleDownloadByID`.
- **Conditional GET / ETag** — strong `ETag: "<file-uuid>"` (files are immutable:
  `File` = pk id + created_at + soft deleted_at, no version). `If-None-Match` ->
  `304` with **no SDK call at all** (skips fetch+reconstruct). `HEAD` -> validators
  only, no fetch.
- **Compression** — `edgeserver/compress.go` lazy gzip `ResponseWriter`; decision
  deferred to first body write so it reads the Content-Type the SDK `WithHeaderSink`
  set. Compressible-type prefix allowlist; skips media + already-`Content-Encoding`
  bodies; MinSize buffer; sets `Vary: Accept-Encoding`, weakens ETag.
  `countingResponseWriter` stays innermost so `edge_download_bytes` = wire bytes.
- **Range (limited, edge-only)** — closed `bytes=start-end` only, via stream-and-trim
  (`rangeWriter` discards `[0,start)`, forwards window, stops SDK with sentinel
  `errRangeDone`). `206` + `Content-Range: bytes s-e/*` (total unknown edge-only).
  Reduces OUTBOUND, not inbound. Suffix/open ranges + real total need go-sdk.
- **Ticket parsing** — `edgeserver/ticket.go`: `fileIDFromTicket` /
  `ticketMatchesFileID` extract the file UUID embedded in BOTH ticket layouts
  (coord-minted `[0:16]` per `coord/file/download_ticket.go`; client scheme-2
  `[1:17]` per go-sdk `download.go`). The match check is the cross-object
  CDN-cache-poisoning guard on the canonical route (400 on mismatch).
- Config: `Cache`/`Compression`/`Range` blocks in `config.go`. Metrics:
  `edge_download_not_modified` / `edge_download_range*` / `edge_download_compressed*`
  in `metrics.go`. README section 4 (CDN fronting + security model + Range limits).

## Key facts / gotchas (durable)
- **`DownloadManifest` proto already has `file_id` + `file_size`**
  (`pkg/pb/coord/file/v1/file.proto:323`), but the edge only has the SDK's
  `DownloadByTicket` (resolve+stream) + `WithHeaderSink` (system/user maps only) —
  so edge-only code can NOT get `file_size` without streaming. That is the hard cap:
  no real Range total, no `Content-Length` on full bodies, no inbound saving. Full
  Range needs a small go-sdk `ResolveTicket` + ranged-stream API (deferred follow-on).
- **Files are immutable** -> file-id ETag is strong; `immutable` Cache-Control truthful.
- **Build env:** this treehouse worktree lacked `../go-sdk` (replace target), so
  `edgeserver` did not build here at all. Fixed by symlinking
  `/home/tuan/.treehouse/depin-b971d9/1/go-sdk -> /home/tuan/work/depin-workspace/go-sdk`.
  Not a repo change; needed for any build/test in this worktree.
- **testplanet e2e is BLOCKED (pre-existing, unrelated):** `internal/testplanet`
  fails to build — `client.Client.CreateContractWithPlacement` now wants a
  `map[string]string` (go-sdk ahead of depin; the pending go-sdk finalize). So the
  planned e2e (`edgeserver_metadata_test.go` pattern) can't run. Covered instead with
  thorough in-package unit tests (ticket parse, range window, gzip round-trip,
  conditional 304/HEAD, integrity 400, 416) — all green, `go vet`+`gofmt` clean.
- Uncommitted (no-auto-commit rule). Plan archived at
  `Projects/depin/plans/2026-07-19-edge-outbound-bandwidth.md`.
