# m3u8 import breaks on AIOZ Stream's own master playlists (demuxed audio)

Date: 2026-09-10. Branch `feat/migrate-with-m3u8` (== origin/stag, 0 diff).

## Fact
`MediaService.generateM3U8` (internal/app/media/service.go:1384) emits **demuxed audio**:
`#EXT-X-MEDIA:TYPE=AUDIO` alternatives plus `AUDIO="audio"` (shared) or `AUDIO=<qualityId>`
(per-quality) on every video variant, and `SUBTITLES="subs"` when captions exist.

The importer refuses exactly that: `checkNoDemuxedAudio` (internal/utils/hlsimport/parse.go:224)
returns `ErrDemuxedAudio` for any variant with `Audio != ""` or an AUDIO alternative.
So an m3u8 URL copied out of AIOZ Stream is rejected at create time by `ValidateSource`.

Reproduced with a synthetic AIOZ-shaped master fed to `ParseMaster`: err = "playlists with
separate audio renditions are not supported", `errors.Is(err, ErrDemuxedAudio) == true`.

## Other gaps found in the same pass
- Audio-only media (`media.Type == AudioType`) masters carry no RESOLUTION -> `ErrNoResolution`.
- `SUBTITLES` alternatives are dropped silently; captions are not imported.
- Write side is fine: hls_import.go:350 stores the full CODECS string in `MediaQuality.VideoCodec`
  and deliberately leaves `AudioConfig`/`AudioPlaylistId` unset (hls_import.go:351), which matches
  the muxed branch of generateM3U8.

## Fixed 2026-09-10 (full demuxed support)
- `classifyAudioGroups` (parse.go) matches #EXT-X-MEDIA renditions to variants by reference
  count: >=2 refs -> `Plan.SharedAudio` (own MediaQuality), 1 ref -> `VariantPlan.Audio`
  (second playlist file on the same quality), 0 refs -> dropped, never fetched.
  New errors `ErrPartialAudioGroup` / `ErrMultipleAudioRenditions` replace `ErrDemuxedAudio`.
- **The decoder pools alternatives per variant**: grafov/m3u8 attaches every #EXT-X-MEDIA to
  whichever variant follows it and then drains the list, so a group map must be built from the
  union of all `variant.Alternatives` - never from one variant.
- A master with a single video variant makes a shared group indistinguishable from a
  per-variant one; it classifies as per-variant. Tests need two variants to hit the shared path.
- `VariantPlan.VideoCodecs` = CODECS minus the audio entry. generateM3U8 re-appends the audio
  codec for a variant in a group, so storing the full string in `MediaQuality.VideoCodec`
  double-emits it.
- Codec vocabulary split, mirroring video: `AudioConfig.Codec` is normalized ("aac"),
  `MediaQuality.AudioCodec` is the raw HLS string ("mp4a.40.2") and is what the master
  advertises. `domain.DefaultSharedAudioCodec` is the fallback for transcoded media, which
  never records one.
- Audio bitrate is measured (segment bytes * 8 / duration): no HLS tag carries it, and
  generateM3U8 names the alternative `audio-%dKbs` from it.
- Serving change: generateM3U8's shared branch emits the shared quality's real `AudioCodec`
  instead of the hardcoded `mp4a.40.2`.
- Still out of scope: importing SUBTITLES renditions, and audio-only masters (no RESOLUTION).

Related: [[hls-serving-contract]], [[per-video-storage-contract]].
