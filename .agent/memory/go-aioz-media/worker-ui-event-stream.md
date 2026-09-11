---
type: decision
tags: [go-aioz-media, worker, ui, polling, superseded]
created: 2026-09-04
updated: 2026-09-08
agent: main
---

**SUPERSEDED 2026-09-08: the WebSocket event stream was REMOVED. The UI polls.**
See [[worker-process-states]] for what replaced it.

The worker exposed a push-only WebSocket at `GET /ui/events` plus an in-process
event bus (`worker/events`). On 2026-09-08 the user chose a polling UI instead,
so the transport and every publisher were deleted:

- deleted: `worker/rest/ws.go`, `ws_test.go`, `worker/events/bus.go`,
  `bus_test.go`, and the emit sites in `task.go` (`emitStepEvent`),
  `step-transcode.go`, `media.go` (3), `sampler.go`.
- `worker/events` survives as a leaf holding only the process/state vocabulary
  and `ProcessStateData`.
- `sampler.go` kept sampling + history + `LastSample` (the hub traffic report
  needs it) and lost all publishing, including `SetMuted`.
- All of this is recoverable from commit `778281e` if the decision reverses.

**What polling cost, stated at the time:** events that begin and end between two
polls are invisible; `file.received` has no REST equivalent; completed tasks
linger only 12s in `/ui/task_status` so a slow poller misses completions. The
user accepted these knowingly.

The UI now polls `/ui/process_state` (1s), `/ui/task_status` (1s) and
`/ui/speed_history` (2s).
