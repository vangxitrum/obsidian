---
type: decision
tags: [go-aioz-media, worker, ui, events, testing, dev-tooling]
created: 2026-09-07
agent: main
---

`aiozworker start --ui-fake-events` (hidden flag) publishes a **synthetic**
`/ui/events` stream so the desktop UI can be developed without a worker that is
actually storing or transcoding.

- Driver lives in `worker/rest/fake_events.go` (`fakeEventDriver`), started next
  to the sampler in `worker/rest/root.go`. No changes to `worker/events` (leaf
  stays leaf) and none to `worker/task-manager`.
- **Events only, by decision.** The driver creates no `StoringRecord`, so
  `GET /ui/task_status` stays empty and `/ui/speed_history` keeps returning real
  (near-zero) data while it runs. Live event UI is testable; the task table is
  not.
- Fake mode calls `speedSampler.SetMuted(true)` (new in `worker/rest/sampler.go`).
  Muting suppresses only *publishing* - sampling, history and `LastSample` keep
  running, so the hub traffic-report loop still sends real numbers and the gauges
  keep their single reader.
- The driver mirrors `Task.emitStepEvent` exactly: `Type: "store"`, `Status`
  strings from `TaskStatus.String()` ("Running"/"Success"/"Failed"/"Canceled"),
  and transcode steps paired with `transcode.*` events. Deviating would show the
  UI a shape it never sees in production.
- 3 slots loop a store lifecycle (download -> transcode -> upload). Every 3rd
  cycle fails in transcode, every 5th cancels in upload, so error states are
  reachable without provoking a real failure.
- Gated on `Bus.HasSubscribers()` at every emit, same as the real emit sites.

**Why:** a real worker needs keystore + hub gRPC + chain + ffmpeg, which is a
heavy prerequisite for GUI work, and real failures are hard to provoke on demand.

**How to apply:** run `./scripts/ui-dev-worker.sh` (builds, creates a throwaway
home + key under `.ui-dev/`, starts on port 31317). The equivalent long form:

    aiozworker keytool new --home <home> --save-priv-key <key> </dev/null
    aiozworker start --home <home> --priv-key-file <key> \
        --laddr tcp://localhost:31317 --ui-fake-events \
        --no-auto-update --single_process=false

As of 2026-09-08 `--ui-fake-events` DOES run fully offline. `rs.Start` reads the
flag once into `uiFakeMode` and seven guards skip: `GrpcClient.Start()`,
`GetHubInfo` + `geolocation.Start` + the hub-info/auto-update loop, the
`GetWorker` registration lookup, `GetHubPubKey`, the traffic-report loop, the
hub-dependent task bootstrap (`UpdateTasks`, `UpdateVerifyingStoringContract`,
`UpdateAssignedContract`), and the public listener / ssh tunnel. Every guard only
ever SKIPS work, so the normal path is unchanged by construction - verified by
running both paths.

`FileManager.Start()` and `TaskManager.Start()` are deliberately KEPT in fake
mode: neither touches the hub (one opens a gate, one is a no-op), and skipping
them nil-pointers `/config/get_storage_config`.

**Trap this exposed:** with the hub skipped, `GrpcClient.conn` and every
per-service client stay nil, so any handler calling the hub PANICS rather than
erroring - `/worker/balance` did. `Start()` dials with `grpc.WithBlock()` and
returns before assigning the sub-clients, so "call Start and ignore the error"
does not help. Fix was `GrpcClient.IsConnected()` (conn != nil), checked by
`getWorkerBalanceHandlerFn` and `liveWithdrawals`, both returning 503. Any other
hub-calling worker route still panics in fake mode if exercised - add the same
check when one comes up. Never extend it to feed `LastSample` while pointed at a real
hub - that would report fake traffic upstream.

Status as of 2026-09-07: implemented on `feat/support-new-ui`, tests green
(`-race`), **deliberately not committed** at the user's request.

**Swagger now exists for the worker** (added 2026-09-08, uncommitted):
`worker/rest/openapi.json` (hand-written OpenAPI 3.0, embedded via `go:embed`)
served by `worker/rest/swagger.go` at `/swagger` (UI) and `/swagger.json`.
Swagger UI is loaded from unpkg, so that page needs internet even though fake
mode is otherwise offline; vendor `swagger-ui-dist` if that matters. It covers
the UI-facing local routes only - not `/debug/*`, `/testnet_migration/*`,
`/worker/register`, or the public delivery routes. No swaggo, so the spec is
hand-maintained and can drift.

`bin/ui-dev-viewer.html` is a single-file dev page (open via `file://`, the WS
origin check allows the `file:` scheme and CORS is `*`) showing the live speed
chart, task progress, withdrawal paging and a raw frame log.

Historical note - **there was no Swagger before this.** `RestServer.RegisterSwaggerUI()` exists
(`worker/rest/root.go`) but its call is commented out, and the only swagger specs
in the repo belong to `uplink/` and `cmd/upload_test/`. The `/ui/*` routes are
documented by hand in `bin/README-ui-events.md` instead - note `bin` is
gitignored, so that file is untracked.

Live `/ui/*` routes are only `events`, `speed_history`, `task_status`,
`node_info`; `worker_info`, `worker_status`, `operator_info` and `hwaccel_info`
are commented out in the router and 404.

See [[worker-ui-event-stream]] and [[worker-ui-telemetry-gotchas]].
