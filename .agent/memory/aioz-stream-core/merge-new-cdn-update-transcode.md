---
type: fact
tags: [merge, transcode, worker, hls, av-sync, dead-code]
created: 2026-08-26
agent: main
---

Fixed the fallout of merge c0a3847 (`feat/update-transcode` into `feat/new-cdn`), which
git resolved cleanly but left the tree broken and one feature silently dropped.

**1. Build break.** `feat/new-cdn` widened `PathManager.Upload` to
`Upload(pl *models.Playlist, contractID string)`; `feat/update-transcode` had meanwhile
moved the call site out of `handleJob` (which had `job`) into the new
`Worker.processPlaylist`, which did not. Fix: thread `contractID` as a parameter of
`processPlaylist`, passed from `process` as `job.ContractId` - the same value the
pre-merge call used.

**2. Silent regression: A/V sync offset dropped.** `VideoConfig.StartTime` /
`AudioConfig.StartTime` still arrive over grpc and are carried through `models`, but the
`args.go` plan builder that replaced `generateCmd` never emitted `-initial_offset`, so a
rendition cut from a non-zero point played out of sync against the rest of the ladder.
Restored as `(*buildContext).initialOffsetArgs`, appended after `outputHygieneArgs`
(i.e. before the output URL). Gated on `SegmentType == HlsType`: `-initial_offset` is an
HLS muxer option and the mp4/dash muxers abort on it. Video StartTime wins over audio's,
matching the old behaviour where the video append came last. Regression coverage in
`internal/core/worker/start_offset_test.go` (3 tests, reproduced failing first).

**3. Dead code from the old side, deleted.** `internal/core/worker/probe.go`
(`probeIsHDR`, `probeIsHighBitDepth`, `buildVideoFilterChain` - the fix recorded in
[[highbitdepth-sdr-profile-crash]]) had no non-test caller after the refactor. Its
behaviour is fully superseded: `models/probe.go` has `Stream.IsHDR/BitDepth/PixFmtInfo`,
`args.go` has `tonemapChain` + `outputPixFmt` + `videoProfile`, and the new
`TestTranscodeCorpus` transcodes `hdr10_bt2020` and `ten_bit_hevc` end to end asserting
8-bit 4:2:0 output - stronger coverage than the old tests. Also deleted
`generate_cmd_test.go` and `repro_audio_test.go`, which failed to compile against the
removed `generateCmd`/`Playlist.CpuCmd` API; the audio-only case they guarded is now the
`audio_only` fixture in the corpus test.

**4. Second silent regression: per-playlist `master.m3u8` gone.** `HlsPlaylistUploader.
Upload` (new-cdn side) opened `<output>/<playlistId>/master.m3u8` to recover the RFC 6381
codec string and BANDWIDTH. The transcode branch dropped `-master_pl_name` from
`SegmentConfig.GenerateSegmentConfig` - deliberately, with a comment saying "nothing reads
a per-playlist master", which was false. Every HLS upload therefore failed with
`open storage/<job>/output/<pl>/master.m3u8: no such file or directory`, *after* the
segments had already been pushed to storage. Fixed by deleting the read, not by
re-adding the flag: `preparePlaylist` already sets `VideoCodec`, `AudioCodec`,
`OutputWidth/Height` and `VideoConfig.Bandwidth` from the encoder settings before the
encode runs. The one field it did not set, `AudioConfig.Bandwidth` (only used by
audio-only renditions - alternatives carry no BANDWIDTH attribute, but the API response
does), now comes from `AudioConfig.Bitrate`, falling back to the new
`models.DefaultAudioBandwidth` (128000, the bps twin of `models.AudioBitrate` "128k").
The existing `TestUploadHls_SegmentsGetDistinctFileIds` masked this by hand-writing a
`master.m3u8` fixture; that write is gone and
`TestUploadHls_NeedsNoPerPlaylistMaster` plus
`TestPreparePlaylistResolvesRenditionMetadata` now guard both halves.

**5. Third silent regression: every progressive MP4 rendition rejected.**
`ProbeResult.verifyVideo` compares `VideoConfig.Width/Height` against the probed source
and fails with `ConfigMismatchErr` on a mismatch. The upstream API fills that field with
the *source* analysis for HLS renditions but with the ladder rung's *target* size for MP4
renditions, so a 480p MP4 of a 1080p source was rejected with `playlist "..." declares
854x480 but the source is 1920x1080` before `preparePlaylist` ever ran - while the 480p
HLS rendition of the same job succeeded. Every `type='mp4'` playlist failed from
2026-08-14 (when source verification landed) onward, at every resolution; the user only
noticed at 480p because that job registered only 480p. Fixed with
`declaresRenditionTarget` in `models/probe.go`: a declared size that equals the
resolution's rung (`VideoQualityTranscodingConfigs[pl.Resolution]`, +/-1px) is the
output, not a mismatched source, so it skips the source comparison. Anything matching
neither is still rejected. Covered by `internal/models/verify_playlist_test.go`; MP4
output also now has e2e coverage in `internal/core/worker/mp4_test.go`
(`TestMp4Rendition`, `TestMp4RenditionCorpus`) - the HLS corpus test never exercised it.

**Takeaway for this repo:** after a merge between a CDN/storage branch and a transcode
branch, `go build` is not enough - `go vet ./...` catches the stale _test.go files, and a
grep for fields that are still parsed but no longer read (`StartTime` here) catches
features the refactor dropped without a compile error. Note `go build ./...` fails on the
untracked `postgres_data/` dir (permission denied); use
`go build ./cmd/... ./internal/... ./pkg/...`.

**Known pre-existing failure, unrelated:** `internal/utils/mpd_parser` `TestParsing`
panics on a missing `test.mpd` fixture - broken since well before either branch tip.

**Known dead code, left alone:** `(*buildContext).needsSoftwareFilters` (args.go) is
unused; `videoFilterChain` inlines the same test as `len(sw) > 0`.
