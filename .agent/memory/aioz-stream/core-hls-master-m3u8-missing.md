---
name: core-hls-master-m3u8-missing
description: job-worker (aioz-stream-core) fails every HLS playlist with "open .../master.m3u8: no such file" since 9e7ce9a dropped -master_pl_name
---

Symptom (job-worker log, 2026-09-10): `ERR handle pl` with
`open storage/<jobId>/output/<playlistId>/master.m3u8: no such file or directory`.
Hits every HLS playlist (video and audio), after ffmpeg succeeds and segments are already uploaded.

Cause: aioz-stream-core commit 9e7ce9a (2026-08-26, "reorganize transcoding into acceleration
fallback chains") removed `-master_pl_name master.m3u8` from HLS ffmpeg args
(`internal/models/job.go:257`, comment claims "nothing reads a per-playlist master"). But
`HlsPlaylistUploader.Upload` (`internal/utils/path_manager/path_manager.go:293`) still opens it
to read CODECS/BANDWIDTH. path_manager tests only cover DASH, so nothing caught it.

`preparePlaylist` (`internal/core/worker/worker.go:453`) now already sets VideoCodec, AudioCodec and
VideoConfig.Bandwidth before upload, so the master read is redundant. Gap: audio-only
AudioConfig.Bandwidth was only ever filled from that master read.

Downstream: playlist marked Fail, aioz-stream `handlePlaylist` FailStatus branch deletes the
uploaded files and refunds transcode cost, so media never completes.

Fix (2026-09-10, core branch `fix/hls-upload-master-read`, uncommitted): dropped the master.m3u8
block in HlsPlaylistUploader; preparePlaylist sets audio-only AudioConfig.Bandwidth (Bitrate or
128000). Guard: `internal/core/worker/upload_e2e_test.go` runs prepare -> ffmpeg -> upload against an
httptest CDN (fake must serve /getBalance with deposit_address + set_credit_later=true, since
MustNewCdnHelper calls it).

Verified live 2026-09-10: job 448edb45 on the rebuilt worker -> both playlists "handle pl done";
aioz-stream media done, 240p quality avc1.420028 bw 1000000, audio quality mp4a.40.2 bw 128000.

Local deploy gotcha: docker-compose-worker.yml mounts host `./bin/w3stream-worker` into job-worker,
so a code fix needs `make build-worker` AND `docker restart job-worker`; restart alone reruns the
old binary. Pre-existing unrelated failure: path_manager TestUploadDashOutput panics (Upload(nil)).

Consumer side check (2026-09-10): no aioz-stream change needed. `handlePlaylist` copies pl.VideoCodec
(video pl), pl.AudioCodec (audio pl), and Bandwidth (VideoConfig else AudioConfig) into the quality row;
proto tags match core (codecs 10/11, video bw 11, audio bw 9). HLS video + audio go out as separate
playlists; in local DB they always land in separate quality rows (`default-audio`), so the
last-writer-wins Bandwidth if/else in handlePlaylist is latent only. `generateM3U8` reads Bandwidth
only for video variants; audio quality bandwidth is unused. Imported rows (no job_id) have audio
bandwidth 0 from hls_import, unrelated to core. `m3u8_helper.MergeM3U8Files` has no callers.
