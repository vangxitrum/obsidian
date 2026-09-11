---
type: fact
tags: [debugging, docker, postgres, worker, logs]
created: 2026-08-28
agent: main
---

How to investigate a failed transcode job in the local docker stack, learned while
diagnosing the MP4-rendition failure in [[merge-new-cdn-update-transcode]].

**The logs rotate and are short.** `docker logs job-worker` / `job-server` only went back
a few hours (both cut at the same timestamp), so the error text for a job that failed
overnight was simply gone. Check `docker logs <c> --timestamps | head -1` first to see how
far back the window reaches before assuming the job produced no output. Rotated files
under `/var/lib/docker/containers/<id>/` need sudo, which is not available here.

**The DB does not store the failure reason.** `models.Playlist.FailureReason` is not a
column on `playlists` - only `status`. So the reason has to come from the logs or from a
local reproduction.

**Useful queries** (container is `job-db`, not `aioz-stream-db`; creds in `server.env`):
```
docker exec job-db psql -U admin -d w3streamcore_job -c \
  "select type, segment_segment_type, status, count(*) from playlists group by 1,2,3;"
docker exec job-db psql -U admin -d w3streamcore_job -c \
  "select id, resolution, type, segment_segment_type, segment_duration, status,
          video_width, video_height, video_bandwidth
   from playlists where job_id='<jobId>';"
```

**How far a playlist got, without logs:** `video_bandwidth` is 0 until `preparePlaylist`
fills it from the rung (`MaxBitrate*1000`, e.g. 2500000 for 480p). A failed playlist with
bandwidth 0 failed *before* `preparePlaylist` - i.e. in `VerifyPlaylist` or
`newBuildContext` - which is what pinned the MP4 bug to the verify step. A sibling
playlist of the same job that succeeded rules out job-wide causes (contract id, source
download, storage).

Worker `storage/<jobId>` is wiped after the job, so there is nothing to inspect on disk
afterwards.
