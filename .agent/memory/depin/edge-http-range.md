---
type: decision
tags: [edgeserver, http-range, 206, content-range, go-sdk, download, range]
created: 2026-07-21
agent: main
---

**Phase 3 (final) of ranged downloads: the edge serves true HTTP Range.** Completes
[[coord-getdownloadinfo-range]] (Phase 1) + [[gosdk-ranged-download]] (Phase 2). Uncommitted
(no-auto-commit). Edge unit tests + a real testplanet e2e green; depin `go build ./...` clean.

## What shipped
- **go-sdk `download.go`**: `ResolveTicketRange(ctx, ticket, offset, length) (*filepb.DownloadManifest, error)`
  - thin: decode ticket + `GetDownloadInfoByTicket` with the range, return the manifest (carries
  whole-file `FileSize` + `SystemHeaders`/`UserMetadata`), coord error returned UNWRAPPED so the edge
  can `status.Code(err)`. Refactored `DownloadByTicketRange` onto it.
- **edge `handler.go`**: replaced the Phase-1 stream-and-trim `serveRange` (+ `rangeWriter`/
  `errRangeDone`/`parseClosedRange`, all removed) with:
  - `parseRange` -> `rangeSpec{kind: closed|open|suffix,...}` (single-range; rejects multi/malformed).
  - `serveRange`: resolve the window via `ResolveTicketRange`, set real `206` +
    `Content-Range: start-end/total` + `Content-Length`, metadata headers from the manifest, stream
    via Phase-2 `DownloadManifestRange`. closed/open = 1 resolve; **suffix `-N` = 2** (a `{0,1}` size
    probe, then the real window).
  - `serveHead`: resolve size (`{0,1}`) -> `Content-Length` + metadata, no body.
  - `writeResolveError`: gRPC status -> HTTP (`OutOfRange`->416, `NotFound`->404,
    `PermissionDenied`->403, `InvalidArgument`->400, else 500).
- **edge `server.go`**: `NewFromClientConfig(client, cfg, logger)` - config-taking constructor
  (zero `Config` from `NewFromClient` leaves Range/compression OFF, so tests/embedders need this).

## Gotchas / decisions (durable)
- **HEAD and any valid range now hit the SDK** (they resolve). The old nil-client edge unit tests for
  HEAD/valid-range no longer work -> moved to the e2e; kept nil-client unit tests only for the paths
  that short-circuit before any resolve (304, integrity/bad-input 400, MALFORMED-range 416 with
  FallbackFull=false). `parseRange` now ACCEPTS suffix/open, so the "unsatisfiable 416" unit test had
  to switch to a malformed range (`bytes=5-2`).
- **Empty (0-byte) file + HEAD/range**: the `{0,1}` probe fails coord's `OutOfRange` (offset 0 >=
  size 0), so HEAD on a 0-byte object returns 416 (rare; a future coord metadata-only resolve would
  fix it). Range responses are identity (no gzip), per HTTP semantics.
- Range responses set `Content-Type` from the manifest (metadataHeaderSink) before `WriteHeader(206)`
  - the old stream-and-trim relied on the SDK header sink mid-download.

## Tests
- edge unit: `edgeserver/download_test.go` `TestParseRange` (closed/open/suffix + malformed).
- e2e: `internal/testplanet/edge_range_test.go` `TestEdgeRangedDownload` - real `Handler()` (built
  with `NewFromClientConfig` + `Range.Enabled`), asserts closed/interior/clamped-past-EOF/open/
  suffix/suffix-larger-than-file windows (body + `Content-Range: s-e/total` + `Content-Length`),
  `416` for offset>=size, and HEAD `Content-Length==size`. Run:
  `DEPIN_TEST_POSTGRES="postgresql://admin:admin123@localhost:5445/hub?sslmode=disable" go test
  ./internal/testplanet/ -run TestEdgeRangedDownload`. PASSED.

## Feature complete
All three phases done: coord range-scoped manifest -> go-sdk ranged decode -> edge true 206. Remaining
housekeeping (not a phase): finalize/pin go-sdk off the `../go-sdk` symlink (commit + pseudo-version),
and commit the depin changes. Related: [[coord-getdownloadinfo-range]], [[gosdk-ranged-download]],
[[edge-outbound-bandwidth]].
