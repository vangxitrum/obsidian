# Import media from an external HLS (m3u8) URL

## Context

Today the only way to get a video into aioz-stream is the three-step upload
pipeline: `POST /api/media/create` -> N x `POST /api/media/:id/part` ->
`GET /api/media/:id/complete`, which concatenates the parts into a local source
file, ffprobes it, registers a transcode job on the external gRPC job server,
and polls until the transcoder produces and uploads HLS renditions.

We want to accept content that is **already** packaged as HLS. The caller hands
us an `.m3u8` URL; we ingest that manifest and its segments into our own storage
and database, then forget the origin. No transcode is performed - the remote
renditions are mirrored byte-for-byte, and the qualities we record are derived
from the manifest itself.

Because the server will fetch caller-supplied URLs, this must not become an SSRF
hole or an unbounded-download hole. Hence explicit URL validation and explicit
size/count/duration limits.

## Decisions (confirmed with the user)

| Question | Decision |
|---|---|
| Ingest mode | Mirror as-is + rewrite playlists. No ffmpeg, no transcode job. |
| Execution | Async worker; the request only validates and returns. |
| Size limits | Manifest file size, per-segment size, segment count / total duration. |
| Manifest shapes | Master **or** media playlist; **VOD only** (reject live). |
| API shape | Extend `POST /api/media/create` with an optional `source_url`. |
| Billing | Storage usage logs only; `TranscodeCost = 0`, no transcode transaction. |
| Metadata | Duration from `EXTINF` sums; **no** thumbnail generation. |
| SSRF policy | Reject loopback/RFC1918/link-local/metadata IPs, re-check every redirect, plus an optional configurable host allowlist. |

## The key discovery that shapes the design

Playback already works entirely off DB rows - nothing on the serving path knows
or cares that a transcoder produced the files:

- `MediaService.generateM3U8` (`pkg/v1/services/video.service.go:1386`) builds the
  master playlist on the fly from `media.MediaQualities`. A quality is included
  only if `Status == done`, `Type == hls`, `VideoConfig != nil`, and it owns a
  `MediaQualityFile` whose `CdnFile.Type == models.CdnVideoPlaylistType`. The
  variant URI is
  `fmt.Sprintf(models.PlaylistUrlFormat, models.BeUrl, q.Id, playlistFile.Offset, playlistFile.Size, models.VideoType)`.
- `MediaService.GetMediaContent` (`video.service.go:1735`) serves
  `/api/media/vod/:qualityId/:filename?range=<off>,<size>&type=&index=N`. It
  finds the `CdnFile` by `(Type, Index)` and reads bytes at `(Id, Offset, Size)`.
  For a playlist it decodes with `m3u8.DecodeFrom` and rewrites `Map.URI` and
  every `segment.URI` to `{BeUrl}/api/media/vod/{qualityId}/{segment.URI}`
  (`video.service.go:1893-1920`), appending `.ts`/`.m4s` when the part before
  `?` has no extension.
- Storage has **no paths or keys**. `StorageHelper.Uploads` packs many files into
  one `fileId` and returns each member's `Offset`/`Size`
  (`internal/utils/storage/cdn.go:274`); those become
  `CdnFile{Id, Offset, Size, Index, Type}` rows via `models.NewCdnFile`.

So the import worker's entire job is: upload the segments, then write exactly the
rows that `handlePlaylist` (`video.service.go:3578`) writes after a transcode.

### The round-trip contract (most important detail)

`SegmentUrlFormat` is `"%s/api/media/vod/%s/%s"` - it contributes **no** query
string. `GetMediaContent` 400s without a `range` param. Therefore the stored
variant playlist must already carry the query string in each segment URI:

Stored (what we upload):

```
#EXTINF:6.000,
segment0.ts?range=0,317184&index=0
```

Served (after `GetMediaContent` rewrites it):

```
#EXTINF:6.000,
https://api.example/api/media/vod/<qualityId>/segment0.ts?range=0,317184&index=0
```

`index` selects the `CdnFile` row; `range` supplies offset+size. Keeping a real
`.ts`/`.m4s` extension on the name means the extension-append branch is a no-op.
The commented-out `internal/core/cdn_handler.go:275` shows this same
`?range=%d,%d&index=1` convention historically.

## Implementation

### 1. Models and constants

- `internal/models/variable.go`
  - add `ImportingStatus = "importing"` next to the other statuses (~`:48`), and
    add it to `ValidMediaStatus` (`:463`) so clients can filter on it.
    A new status is needed rather than reusing `waiting`/`transcoding`: the
    `uploadWaitingMediaSource` cron (`internal/cron/cron.go:141`) would try to
    open a non-existent local source file for `waiting` media, and
    `StartWatchPlaylist` would log an empty-JobId warning every 5s.
- `internal/models/media_import.go` (new) - `MediaImportTask` + repository
  interface, modelled on `AiGenerationTask` (`internal/models/task.go:31`):
  `Id, MediaId, UserId, SourceUrl, Status (pending|processing|done|fail),
  Error string, Attempts int, CreatedAt, StartedAt, UpdatedAt`.
  Task state lives in its own table rather than on `Media` so retries, attempt
  counts and the failure reason survive without widening `Media`.
- `models.Media.Source` (already exists, currently unused) stores the origin URL
  for provenance.

### 2. Repository

- `pkg/v1/repositories/media_import.repository.go` (new) - same shape as
  `pkg/v1/repositories/quality.repository.go`: constructor takes `db, init bool`
  and calls `AutoMigrate(&models.MediaImportTask{})`. There is no migration tool
  in this repo; AutoMigrate in the repo constructor is the convention.
  Methods: `Create`, `GetTasksByStatus`, `ClaimNextPending` (a
  `status='pending' -> 'processing'` update guarded by the current status, so two
  processes cannot claim the same task), `Update`, `GetByMediaId`.

### 3. Config

`internal/config/config.go` (`AppConfig`, plus `env-example/app.env`), following
the existing SCREAMING_SNAKE `mapstructure` convention:

| Key | Default | Purpose |
|---|---|---|
| `HLS_IMPORT_ENABLED` | `false` | Feature flag. |
| `HLS_IMPORT_MAX_MANIFEST_SIZE` | `2MiB` | Cap on any single `.m3u8` body. |
| `HLS_IMPORT_MAX_SEGMENT_SIZE` | `200MiB` | Cap on any single segment. |
| `HLS_IMPORT_MAX_SEGMENTS` | `5000` | Per variant. |
| `HLS_IMPORT_MAX_VARIANTS` | `10` | Per master playlist. |
| `HLS_IMPORT_MAX_DURATION` | `4h` | Sum of `EXTINF` per variant. |
| `HLS_IMPORT_HTTP_TIMEOUT` | `30s` | Per outbound request. |
| `HLS_IMPORT_CONCURRENCY` | `4` | Concurrent segment downloads within a task. |
| `HLS_IMPORT_ALLOWED_HOSTS` | empty | Comma-separated; when non-empty, only these hosts may be fetched. |
| `HLS_IMPORT_MAX_PACK_BYTES` | `64MiB` | Batch cap for `Uploads` (see below). |

### 4. Safe outbound fetcher

`internal/utils/hlsimport/fetcher.go` (new). Model it on the tuned transport in
`internal/utils/storage/cdn.go:118` (`handleRequest`) - that is the only existing
outbound client with sensible timeouts; the ones in `internal/utils/stream` have
none, so do not copy those.

- Require `http`/`https`.
- `net.Dialer.Control` hook rejects the resolved IP when it is loopback,
  private (RFC1918), link-local, unique-local, unspecified, or
  `169.254.169.254`. Doing the check in `Control` (post-resolution, pre-connect)
  closes the DNS-rebinding window that a resolve-then-dial check leaves open.
- `CheckRedirect` re-runs host allowlist validation on every hop and caps hops at 5.
- `FetchManifest` reads through an `io.LimitedReader` of
  `MaxManifestSize + 1` and errors if the limit is hit.
- `FetchSegment` streams to a temp file, rejecting on `Content-Length` when
  present and on the limit while copying when it is not.

### 5. Manifest parsing

`internal/utils/hlsimport/parse.go` (new), using the existing
`github.com/grafov/m3u8` dependency:

- `m3u8.DecodeFrom(body, true)` (strict).
- Master playlist: cap `len(Variants)`, resolve each variant URI against the
  master URL, fetch and parse each variant.
- Media playlist supplied directly: treat it as a single variant with no
  `EXT-X-STREAM-INF` attributes.
- **VOD only**: reject when `!MediaPlaylist.Closed` (no `EXT-X-ENDLIST`) or
  `MediaType == m3u8.EVENT`.
- **Reject** `EXT-X-KEY` with a method other than `NONE` - mirroring encrypted
  segments without mirroring the key would produce unplayable media.
- **Reject** segments carrying `EXT-X-BYTERANGE`: our serving layer already uses
  the `range` query param for its own packed-storage offsets, so a source-level
  byte-range would collide.
- Enforce segment count and summed `EXTINF` duration per variant.

Variant -> `MediaQuality` mapping:

| Source | Target |
|---|---|
| `RESOLUTION=WxH` | `VideoConfig.Width/Height`; `Resolution` and `Name` = `"<H>p"` |
| `BANDWIDTH` | `Bandwidth`, and `VideoConfig.Bitrate` |
| `CODECS` | split into video/audio; `VideoCodec`, `AudioCodec`, `AudioConfig` |
| segment extension | `ContainerType`: `.ts` -> `mpegts`, `.m4s`/`.mp4` -> `fmp4` |
| always | `Type = hls`, `Status = waiting` -> `done`, `TranscodeCost = 0` |

Edge cases: when `RESOLUTION` is missing, ffprobe is not available in this path,
so fall back to `Name = "source"` and derive nothing - but a quality with a nil
`VideoConfig` is skipped by `generateM3U8`, so instead **require** `RESOLUTION`
on video variants and reject the import with a clear message when it is absent.
Non-standard heights (not in `models.ValidMediaQualities`) are snapped to the
nearest standard label for `Name`/`Resolution` while `VideoConfig.Width/Height`
keep the true values - `ValidMediaQualities` gates user-declared qualities at
create time, and imported qualities bypass that validation entirely. Audio-only
manifests map onto `media.Type = audio` and the `CdnAudioPlaylistType` /
`CdnAudioContentType` file types, which `generateM3U8`'s `AudioType` branch
already handles.

### 6. Import service (the worker)

`pkg/v1/services/hls_import.service.go` (new), started from `cmd/http/main.go`
next to `go mediaSummaryService.Run(ctx)` (`main.go:131`) and following
`MediaSummaryService.Run` (`pkg/v1/services/media_summary.service.go:133`): a
ticker that claims pending tasks, with a staleness cutoff that returns tasks
stuck in `processing` to `pending` (bounded by `Attempts`, then `fail`).

Per task:

1. Load the media; skip if deleted. Set `media.Status = importing`.
2. Fetch + parse the manifest (limits and SSRF checks as above).
3. For each variant, in order:
   a. Create the `MediaQuality` row (`qualityRepo.Create`).
   b. Download segments to `{INPUT_STORAGE_PATH}/{mediaId}/import/{qualityId}/`,
      up to `HLS_IMPORT_CONCURRENCY` at a time.
   c. Upload in batches via `storageHelper.Uploads`. **This call buffers the
      whole multipart body in a `bytes.Buffer`** (`cdn.go:274-330`), so batches
      must be capped by `HLS_IMPORT_MAX_PACK_BYTES`, not by file count. Each
      batch returns one `fileId` with per-member offsets.
   d. Assign `CdnFile.Index` from a **single counter across the whole quality**,
      not per batch - segments spanning several packs still need a unique
      `(Type, Index)` because that pair is the lookup key in `GetMediaContent`.
      Different `fileId`s per batch are fine; `Id` is stored per row.
   e. Build the variant `MediaPlaylist` with
      `m3u8.NewMediaPlaylist(0, n)` (winsize 0 = VOD), append each segment with
      URI `"<name>?range=<offset>,<size>&index=<i>"`, carry `EXT-X-MAP` through
      the same way when present, then `Close()` to emit `EXT-X-ENDLIST`.
      Put the query string inside `seg.URI` and leave `MediaPlaylist.Args`
      empty - the encoder appends `Args` identically to every segment
      (`writer.go:697`), but each segment needs its own range, and it writes
      `seg.URI` verbatim (`writer.go:696`), which is what we want.
   f. `storageHelper.PackUploadByte` the playlist bytes -> `CdnFile` with
      `Type = video_playlist`, `Index = 1`.
   g. Write all `MediaQualityFile` + `CdnFile` rows via `qualityRepo.CreateFile`,
      exactly as `handlePlaylist` does (`video.service.go:3627`).
   h. Set `quality.Status = done`, `TranscodedAt = now`, persist.
   i. Write a storage `UsageLog` via
      `(&models.UsageLogBuilder{}).SetUserId(...).SetStorage(total).SetIsUserCost(true)`,
      mirroring `video.service.go:3658`. No transcode log, no
      `paymentClient.CreateTransaction`.
4. Delete the temp download directory.
5. Set `media.Size` = total bytes, write a `MediaFormat` row with the summed
   `EXTINF` duration (so `media.GetMediaDuration()` works), `media.Source` =
   origin URL, `media.Status = done`.
6. Publish the same webhooks the normal pipeline does, so API consumers see no
   difference: `EventFileReceived` on start and `EventEncodingFinished` on
   success / `EventEncodingFailed` on failure (`internal/models/webhook.go:17-21`),
   via `s.callWebhookCh.Publish`.

On failure: mark the task `fail` with the reason, set `media.Status = fail`, and
delete whatever was already uploaded by walking the quality rows written so far -
reuse `deleteMediaQuality` (`video.service.go:4045`) so no orphan `CdnFile`s are
billed. Because each pack upload's `fileId` is recorded on its rows before the
next batch starts, a crash mid-import leaves a fully reconstructable set to clean
up on retry.

### 7. Controller and route

No new route and no auth change: `POST /api/media/create` is already in the
`OnlyUpload` API-key allowlist (`internal/utils/validate/validator.go:56`).

- `internal/controllers/video.controller.go`
  - add `SourceUrl string \`json:"source_url" form:"source_url"\`` to
    `CreateMediaRequest` (`:51`), documented as "import an existing HLS manifest;
    when set, qualities are derived from the manifest and the part-upload flow is
    skipped".
  - in `CreateMediaObject` (`:114`), when `SourceUrl` is non-empty:
    reject when the feature flag is off; reject when `Qualities` is also supplied
    (they are mutually exclusive); run the **synchronous** validation - URL
    shape, scheme, host allowlist, IP guard, fetch the manifest under the size
    cap, parse it, confirm VOD, confirm at least one usable variant, confirm the
    limits - so a bad URL is a fast `400` with a specific message. Then call the
    new service entry point and return `202` with the media object at status
    `importing`.
  - the existing title/description/type/tags/metadata validation runs unchanged
    before this branch. The quality-defaulting block (`:190-325`) is skipped for
    imports.

Keep the branch small: put the validation itself in
`internal/utils/hlsimport` and have the controller call one
`hlsimport.Validate(ctx, url) (*Manifest, error)`. `CreateMediaObject` is already
~400 lines; this adds a guard clause and a call, not another inline block.

- `pkg/v1/services/video.service.go` - add
  `CreateMediaObjectFromUrl(ctx, media, sourceUrl)` next to
  `CreateMediaObjectLiveStream` (`:260`): both funnel into the existing
  `createMediaObject` (`:272`) so secret generation, default player theme and the
  `mediaRepo.Create` all stay in one place. It then writes the
  `MediaImportTask` row in the same DB transaction.

### 8. Small in-scope fix

`deleteMediaQuality` (`video.service.go:4045`) calls `storageHelper.Delete` once
per `CdnFile`. With packed uploads many rows share a `fileId`, so an import of a
few thousand segments would issue a few thousand redundant delete calls. Dedupe
by `FileId` before deleting. This is pre-existing behaviour but imports make it
sharply worse.

## Files touched

| File | Change |
|---|---|
| `internal/utils/hlsimport/fetcher.go` | new - SSRF-safe HTTP client, size-capped reads |
| `internal/utils/hlsimport/parse.go` | new - manifest parse, VOD check, limits, variant mapping |
| `internal/utils/hlsimport/playlist.go` | new - build the stored variant playlist |
| `pkg/v1/services/hls_import.service.go` | new - the worker |
| `pkg/v1/repositories/media_import.repository.go` | new - task repo + AutoMigrate |
| `internal/models/media_import.go` | new - task model + repo interface |
| `internal/models/variable.go` | add `ImportingStatus`, extend `ValidMediaStatus` |
| `internal/config/config.go`, `env-example/app.env` | new `HLS_IMPORT_*` keys |
| `internal/controllers/video.controller.go` | `source_url` field + import branch in `CreateMediaObject` |
| `pkg/v1/services/video.service.go` | `CreateMediaObjectFromUrl`; dedupe in `deleteMediaQuality` |
| `cmd/http/init.go`, `cmd/http/main.go` | wire + start the import service |
| swagger | `make gen-swagger` (note: `swag fmt` also reformats unrelated generated files - check the diff and revert strays) |

## Verification

The repo has almost no test infrastructure (4 test files, all hitting live
remote services, no CI test job), so verification is primarily end-to-end
against the local stack, matching the project rule of preferring end-to-end
checks over unit tests.

1. **Unit, where it is cheap and valuable** - table tests for the pure pieces,
   which need no infrastructure: the SSRF IP classifier (loopback / RFC1918 /
   link-local / `169.254.169.254` / IPv6 forms all rejected, a public IP
   accepted), the variant -> `MediaQuality` mapping, and the round-trip
   `buildStoredPlaylist` -> `m3u8.DecodeFrom` -> the same rewrite
   `GetMediaContent` performs, asserting the final segment URL is exactly
   `{BeUrl}/api/media/vod/{qualityId}/seg0.ts?range=0,N&index=0`.
2. **Local origin** - serve a small VOD HLS tree (master + 2 renditions) from
   `python3 -m http.server` in the scratchpad. Because the SSRF guard rejects
   loopback, run this leg with `HLS_IMPORT_ALLOWED_HOSTS` set to that host and
   an explicit dev-only escape for loopback, or point at a LAN address - decide
   which at implementation time and document it in `env-example/app.env`.
3. **End-to-end** - bring up the local stack (`app.env` + infra containers per
   the project's local-dev notes), then:
   - `POST /api/media/create` with `source_url` -> expect `202` and status
     `importing`.
   - poll `GET /api/media/:id` until `done`.
   - `GET /api/media/:id/manifest.m3u8` -> a master listing both renditions.
   - follow a variant URL -> a playlist whose segment URLs are absolute and
     carry `range`/`index`.
   - follow a segment URL -> the bytes, `curl -o` it and `cmp` against the
     original file from the origin server. **This is the check that proves the
     mirror is byte-exact.**
   - play the master URL in ffplay or a browser HLS player.
4. **Negative cases** - each expects a specific `400`: a non-HLS URL; a live
   playlist (no `EXT-X-ENDLIST`); a manifest over the size cap; a manifest whose
   segment count or duration exceeds the caps; `http://127.0.0.1/...` and
   `http://169.254.169.254/...`; a host outside a non-empty allowlist; a redirect
   from a public host to `127.0.0.1`; an `EXT-X-KEY` encrypted playlist; a
   request supplying both `source_url` and `qualities`.
5. **Cleanup** - `DELETE /api/media/:id` on an imported video, then confirm the
   `cdn_files` rows are gone and `GetUserTotalStorage` drops back.
6. `make build` and `make lint` (golangci-lint v2 is blocking in CI).
