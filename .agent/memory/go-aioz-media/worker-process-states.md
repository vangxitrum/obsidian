---
type: decision
tags: [go-aioz-media, worker, events, ui, process-state]
created: 2026-09-08
agent: main
---

The worker reports three node-level process state machines to the UI through
`GET /ui/process_state`, which the UI POLLS. There is no event stream
([[worker-ui-event-stream]] was removed 2026-09-08).

```
delivery:    idle -> serving -> done -> serving -> done ...
storage:     idle -> retrieving -> storing -> done -> ...
transcoding: idle -> receiving -> encoding -> uploading -> done -> ...
```

**`done` is STICKY** - a process rests there until new work arrives. This is the
key polling adaptation: a state left as soon as it is entered can only be caught
by luck, so `done` persists and is what the UI shows between jobs. `idle` means
"has not worked since start"; a process never returns to it. Pinned by
`TestDoneIsStickyForPolling`.

**The classification rule is the non-obvious part.** The transcoding pipeline
(`worker/rest/media.go` ~1330-1400) is `download -> save-file -> transcode ->
checksum -> commit-transcode -> upload` and is built as **`TaskTypeStore`** -
the same task type as plain storage. So a process CANNOT be identified by task
type; `taskProcess()` scans `Task.Steps` for a `StepTypeTranscode` and only then
falls back to storage. Pinned by `TestTaskProcessClassification`.

- Tracker: `worker/task-manager/process_state.go`, hooked into `Task.Execute`
  (`TaskStarted` / `StepStarted` / deferred `TaskFinished`).
- **Node level, not per file.** That is what makes delivery affordable:
  `/file/{fileId}` runs a task per file AND per segment, so delivery contributes
  only an atomic counter and emits on the `0->1` and `1->0` edges. Serving 25
  files produces exactly one `serving` - pinned by
  `TestDeliveryAggregatesInsteadOfPerTask`. This keeps the original
  deliver-exclusion decision ([[worker-ui-event-stream]]) intact.
- `done` is a transition emitted once before returning to `idle`, not a resting
  state.
- The tracker publishes nothing; it only records state. `ProcessStateData`
  carries `since` (unix seconds) so a poller can render "encoding for 40s".
- Fake mode is now a whole synthetic node in `worker/rest/fake_state.go`: it
  drives the three machines AND fills `/ui/task_status` and `/ui/speed_history`
  (30 min pre-seeded, so the chart is populated on first load). Without this a
  polling UI in fake mode would show an empty task list and a flat chart.

Step->state maps live in `storageStepState` / `transcodingStepState`; a step that
maps to nothing leaves the process where it is rather than inventing a state.

See [[ui-fake-events-flag]] and [[worker-ui-event-stream]].
