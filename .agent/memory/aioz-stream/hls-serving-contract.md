---
type: fact
tags: [hls, m3u8, playback, cdn, storage, gotcha]
created: 2026-09-03
agent: main
---

# The aioz-stream HLS serving contract (and its traps)

Learned while building the "import media from an external m3u8 URL" feature
(branch `feat/migrate-with-m3u8`, plan
`Projects/aioz-stream/plans/2026-09-03-import-media-from-external-hls-url.md`).

Playback is driven **entirely by DB rows** - nothing on the serving path knows a
transcoder produced the files. Anything that writes the same rows gets playback
for free. The rows are `MediaQuality` + `MediaQualityFile` -> `CdnFile`, written
by `handlePlaylist` (`pkg/v1/services/video.service.go:3578`).

Storage has **no paths or keys**: `StorageHelper.Uploads` packs many files into
one `fileId` and returns per-member offsets. A file is addressed as
`(CdnFile.Id, offset, size)`.

## The round-trip contract

`SegmentUrlFormat = "%s/api/media/vod/%s/%s"` carries **no query string**;
`GetMediaContent` (`video.service.go:1735`) merely *prefixes* the stored URI. So
everything the follow-up request needs must already be in the stored playlist:

    s0.ts?range=<offset>,<size>&index=<packIndex>

- `range` is **mandatory** - the controller 400s without it.
- `index` selects the `CdnFile` row and **defaults to 1** when absent. That is
  why the variant playlist's own `CdnFile.Index` must be `1`: `PlaylistUrlFormat`
  omits `index`.
- `type` defaults to `"video"`. An **audio** rendition's stored segment URIs must
  carry `&type=audio` or every segment 500s, because the rewrite drops `type`.

## Traps that cost real debugging time

1. **`#EXT-X-ENDLIST` is load-bearing, not cosmetic.** `m3u8.DecodeFrom` starts a
   playlist at `winsize 8` and only widens it to "show everything" for a playlist
   that is `Closed` or `EVENT` (`reader.go:196,247`); `Encode` loops
   `i < p.winsize || p.winsize == 0` (`writer.go:564`). A stored playlist without
   ENDLIST is **silently truncated to its first 8 segments** when served - video
   stops after ~1 minute. Always `playlist.Close()` before `Encode()`.
   `#EXT-X-PLAYLIST-TYPE:VOD` alone does **not** save you.
2. **`generateM3U8` 500s the whole manifest** if any done/hls quality has
   `AudioConfig != nil` but no `CdnAudioPlaylistType` file. One bad quality row
   breaks playback for *every* rendition of that media. For muxed renditions
   leave `AudioConfig`/`AudioPlaylistId` unset and put the full CODECS string in
   `VideoCodec` (it is assigned straight to `variant.Codecs`).
3. **`ContainerType` must be `mp4`, never `fmp4`, for fMP4.** `GetMediaContent`
   maps only `mpegts`->`.ts` and `mp4`->`.m4s`; `fmp4` falls through to an empty
   extension and `application/octet-stream`.
4. **`CdnHelper.handleRequest`'s retry renames every multipart part to `"file"`**
   (`internal/utils/storage/cdn.go:~213`), destroying the name->offset mapping of
   a multi-file `Uploads`. Always pass `*bytes.Reader` (the retry seeks) **and**
   verify the returned `Object.Name` set matches what you sent; discard the pack
   if it does not. Also: `Uploads` buffers the whole body in memory - batch it.
5. `MediaQuality.Name` must be `"1080p"`-shaped: `GetMediaM3U8` sorts renditions
   by `Atoi(TrimSuffix(Name,"p"))`.
6. `internal/utils/m3u8_helper.MergeM3U8Files` is **dead code** with an inverted
   condition. The live master generator is `MediaService.generateM3U8`.
7. Media status `waiting` is claimed by the `uploadWaitingMediaSource` cron
   (opens a local source file) and `{transcoding,waiting}` by
   `StartWatchPlaylist`. A non-transcode pipeline needs its **own** status, and
   must be added to `Media.IsTranscoding()` or `DeleteMedia` will delete media
   mid-write. Map it to `transcoding` in `NewMediaObject` to keep the public API
   contract unchanged.

See [[per-video-storage-contract]] for the go-sdk/contract side (not active on
this branch - `CdnHelper` is the wired storage impl, `cmd/http/init.go:483`).
