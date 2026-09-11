# go-aioz-media — memory index

Worker/hub media node written in Go. Module `github.com/AIOZNetwork/go-aioz-media`.

- [[worker-ui-event-stream]] — push-only WebSocket at `/ui/events` + in-memory speed history; how to add new events.
- [[worker-ui-telemetry-gotchas]] — snapshot-not-live task records, the mutating `Gauge.LastValue`, per-delivery Tasks, ffmpeg `\r` framing, and the repo's pre-existing build/vet/gofmt baseline.
- [[ui-fake-events-flag]] — hidden `--ui-fake-events` flag publishes a synthetic UI event stream for GUI dev; events only, mutes the sampler.
- [[worker-withdraw-history]] — `/worker/withdraw_history` cursor-paged route; both phases built (uncommitted): fake seed + full hub chain; proto codegen recipe verified.
- [[worker-process-states]] — three node-level process machines (`process.state` + `/ui/process_state`); transcoding is a TaskTypeStore, so classification scans steps.
- [[worker-binary-role-split]] — three worker binaries (aiozworker/aiozstorage/aioztranscode) from one `role.Role`; the ffmpeg embed is the only real size difference, and no hub change was needed.
