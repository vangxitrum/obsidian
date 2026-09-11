---
type: fact
tags: [go-aioz-media, worker, metrics, ui, gotcha]
created: 2026-09-04
agent: main
---

Worker-side telemetry landmines found while building the UI event stream
(2026-09-04). These are non-obvious and cost real debugging time:

- **`Statistic.storingRecords` holds snapshots, not live state.**
  `UpdateStoringTask` (`worker/task-manager/statistic.go`) allocates a *new*
  `StoringRecord` and is called exactly once per step entry from
  `Task.Execute`. Anything read out of the map is frozen at step-start.
  Progress must be read from the live `Step` at read time - that is what
  `StoringRecord.Snapshot()` / `fillProgress()` now do.
- **Completed tasks linger 12s.** `Task.Execute` defers `DeleteStoringTask`
  behind a `time.Sleep(12 * time.Second)` goroutine. Anything that walks the
  record map must skip terminal steps or it emits ghost progress.
- **`utils/metrics.Gauge.LastValue()` mutates the gauge** - it is what rolls the
  accumulation window. Two independent readers change each other's numbers.
  The worker now funnels every read through the single sampler in
  `worker/rest/sampler.go`; the hub-report loop and `/stats` read its cache.
- **`/file/{fileId}` executes a full `Task` per file/segment served**
  (`worker/rest/media.go`, `getFileHandlerFn`). It is the busiest path in the
  worker - never add unconditional work to `Task.Execute` without excluding
  `TaskTypeDeliver` and gating on whether anything is listening.
- **`TranscodeEnv.FfmpegPath()` / `FfprobePath()` re-extract and `os.WriteFile`
  the binary on every call** (`transcode/transcode.go`). Calling them in a loop
  means a disk write per call, and rewriting a binary another transcode is
  executing gives `ETXTBSY` on Linux. Both are now behind a `sync.Once`.
- **`ffprobetool.GetMediaInfo` hardcodes `exec.Command("./ffprobe")`**, which is
  wrong for the worker's extracted binary. Use `GetMediaInfoWithPath`.
- **ffmpeg's stats line ends in `\r`, not `\n`**, so `bufio.Scanner` with
  `ScanLines` yields one giant line at process exit. See
  `transcode/progress.go`.
- `gorilla/websocket` clears the `http.Server` read/write deadlines right after
  `Hijack()`, so the worker's 3600s timeouts do not apply to a WS connection -
  but `netutil.LimitListener` (cap 1000) still does, and that listener is shared
  with the file-delivery routes when no public address is configured.

Repo build baseline: `go build ./...` already fails on `utils/zip/test`,
`utils/reputation-score`, `cmd/log_filter` and `cmd/upload_test` on a clean
tree. `go vet` likewise has pre-existing findings in `worker/rest` and
`worker/task-manager`. Diff against a `git stash -u` baseline before blaming a
change. Several files are also not `gofmt`-clean on purpose (narrower
formatter), so never blanket-`gofmt -w` the package.

See [[worker-ui-event-stream]].
