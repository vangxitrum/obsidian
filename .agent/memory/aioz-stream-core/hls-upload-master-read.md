---
name: hls-upload-master-read
description: HLS uploads failed every rendition because HlsPlaylistUploader still opened a per-playlist master.m3u8 that ffmpeg stopped writing in 9e7ce9a.
type: project
date: 2026-09-10
---

Branch `fix/hls-upload-master-read`.

- Cause: 9e7ce9a dropped `-master_pl_name` from HLS args (`internal/models/job.go`), but `HlsPlaylistUploader.Upload` (`internal/utils/path_manager/path_manager.go`) still opened `<output>/master.m3u8` to read codecs + bandwidth. Error: `open .../output/<pl>/master.m3u8: no such file or directory`, after segments + playlist were already uploaded. Every HLS playlist failed.
- Fix: delete the master read. Codecs/video bandwidth already set pre-transcode in `preparePlaylist` (worker.go). Added audio-only bandwidth there (AudioConfig.Bitrate, else 128000) since that was the only value still sourced from the deleted read.
- Guard: `internal/core/worker/upload_e2e_test.go` `TestProcessPlaylistUploadsHLS` (real ffmpeg + httptest fake CDN, video + audio-only cases). Verified: fails on HEAD 9a7d34c, passes with fix; uploader fix alone still fails audio case on bandwidth assert.
- Most of the worker.go diff is line-wrap noise from the Go formatter hook ([[go-edit-formatter-hook]]).
