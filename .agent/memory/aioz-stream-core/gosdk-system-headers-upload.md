---
type: decision
tags: [go-sdk, cdn, storage, content-type, cache-control, system-headers]
created: 2026-07-13
agent: main
---

Wired `Content-Type` + `Cache-Control` into every `StorageHelper` upload call
(`internal/utils/cdn/cdn.go`) as go-sdk "system headers", so they're cached on the
object at upload time instead of the (not-yet-existing) edge server having to
re-derive them on every download. See [[gosdk-storagehelper]] (in the sibling
`aioz-stream` project folder) for the go-sdk client fix (`UploadParams.SystemHeaders`,
commit `c059e8f` "feat(upload): add system headers and user metadata") this depended
on - that commit landed on go-sdk mid-session while this work was being planned, added
by the user concurrently in another window (same pattern as prior sessions: go-sdk is
a fast-moving local dependency, always re-check `git log -1` before assuming a gap is
still open).

**Design:** `StorageHelper.Upload/Uploads/UploadZip/UploadRaw` each gained a
`systemHeaders map[string]string` param (one map per call, not per-file - every batch
upload shares one content type). Added `cdn.SystemHeaders(contentType, cacheControl
string) map[string]string` builder plus `cdn.ContentTypeMp2T/Mp4/HlsManifest/
DashManifest` and `cdn.SegmentCacheControl` ("immutable, max-age=31536000")/
`cdn.ManifestCacheControl` ("max-age=60") constants. `CdnHelper` (legacy HTTP backend)
accepts but ignores the param - `/packUpload` has no header-store concept.
`MemoryHelper` (test double) records it per stored id via a new `SystemHeadersOf(id)`
accessor, used in `path_manager_test.go` to assert the wiring end-to-end.

**Key finding that shaped the design:** `StorageHelper.Upload`'s `data string` param
is NOT a reliable filename - it's reused as the caller's video/playlist ID at several
call sites (source video upload, mp4 output, master/media playlist, DASH manifest),
while `Uploads`' `files map[string]io.Reader` keys ARE real filenames with extensions
(HLS/DASH segment batches). So content-type could not be derived generically from
`data`; each of the 10 call sites had to pass what it already knew: `video.Mimetype`
for source uploads, a fixed constant for playlists/manifests/mp4 output, segment
extension (`playlistData.Map != nil` → fmp4/`video/mp4`, else TS/`video/mp2t`) for HLS
segments, and `representation.MimeType` (already accurate, straight from the MPD) for
DASH segments.

**Coord-side was already fully built** (`depin` repo, `coord/file/metadata.go`):
allowlisted system headers (`content-type`, `cache-control`, `content-disposition`,
`content-encoding`, `content-language`, `expires`), normalized to lowercase, validated
against header-injection chars, persisted, returned on `DownloadManifest`. go-sdk's
download side already reads them back via `DownloadOption WithHeaderSink`
(`download.go`) - the hook an edge server would use to turn stored metadata into HTTP
response headers. Only the upload-side client wrapper was the gap, and the user closed
it themselves (commit `c059e8f`) while this plan was being drafted.

Scope was confirmed with the user via AskUserQuestion before implementing: cover ALL
upload call sites (not just segments), and set both Content-Type + Cache-Control (not
content-type only).
