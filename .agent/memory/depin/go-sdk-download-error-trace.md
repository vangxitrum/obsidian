---
type: reference
tags: [depin, go-sdk, edge, download, errors, timeout]
created: 2026-08-03
agent: main
---

The edge's `"download failed"` HTTP body is intentionally generic. The useful
cause is preserved in the edge log's structured `error` field. Full-download
errors flow through `edgeserver/handler.go:serveFull` ->
`go-sdk.DownloadByTicket` -> `DownloadManifest` -> `streamSegments` ->
`segmentdownload.Open` / `io.Copy` -> `piecedownload.Manager.Fetch` ->
`workerclient.GetPiece` / eestream. Prefixes identify the stage:
`sdk: resolve download ticket`, `sdk: open segment N`, `sdk: stream segment N`,
or `sdk: close segment N`.

HTTP mapping is only `errors.Is(context.DeadlineExceeded)` -> 504; every other
pre-body failure -> 502. A failure after body bytes causes
`http.ErrAbortHandler` so clients detect truncation.

Important timeout gap: `DownloadByTicket` performs coordinator
`GetDownloadInfoByTicket` before calling `DownloadManifest`, while
`withDownloadTimeout` is created inside `DownloadManifest`. Therefore comments
claiming `WithDownloadTimeout` includes ticket/manifest resolution are currently
incorrect; a stalled coordinator resolve is bounded only by the caller context.
The same gap exists in range resolve (`ResolveTicketRange`).

Workspace caveat: depin's `go.mod` replaces `aioz-depin/go-sdk` with
`../go-sdk`. That SDK working tree has uncommitted download fixes (deadline
classification, 20s keepalive ACK timeout, eestream prefetch/quiescence fix), so
always compare the running edge build hash before treating current local source
as deployed behavior. See [[download-failure-fixes-applied]] and
[[segment-prefetch-quiescence-bug]].
