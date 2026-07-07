---
type: doc
project: depin
tags: [depin, worker, autoupdate, versioncontrol]
created: 2026-07-05
---

# Plan: Worker auto-update — faithful port of storj's storagenode-updater

## Context

Storj storage nodes auto-update via a **separate `storagenode-updater` daemon** (storj/docs/autoupdate.md):
on an interval it polls a version server for `AllowedVersions`, decides per-binary via
`version.ShouldUpdateVersion` (up-to-date / below-minimum-unconditional / HMAC staged-rollout gate),
then downloads → verifies → backs up → swaps → restarts the service out-of-process. It manages
**two** binaries (the node and the updater itself). The version server ramps a per-process rollout
**cursor** over time (`calculateRolloutCursor`, `SafeRate`/day) so a bad release reaches only a
controlled fleet slice.

depin already has strong building blocks but a **divergent** design:
- `pkg/version` — near-complete port of `storj.io/common/version`: `AllowedVersions`/`Processes`/
  `Process{Minimum,Suggested,Rollout}`/`Version{Version,URL}`/`Rollout{Seed,Cursor}`,
  `ShouldUpdateVersion`, `isRolloutCandidate` (HMAC), `PercentageToCursorF`, `getInfoFromBuildInfo`.
  **Reuse this wholesale.** Gaps: `Build` never populated (`info.go init()` commented out → worker
  reports `0.0.0`); `Processes` still has storj names, no `worker`/`worker-updater`.
- `versioncontrol/` + `cmd/versioncontrol` — a gin server that filters binaries by
  OS/arch/location/cluster build-tags; it does **not** serve `AllowedVersions` and has **no**
  rollout cursor. Its `version.AllowedVersions` field (`api/api.go:22`) and `SafeRate`/`RegenInterval`
  config are dead.
- `worker/version/` — an **in-process** self-update client (`go-update` + keytool-signed HTTP). The
  chosen storj model uses an external daemon with rename-swap+restart, so this client is
  **superseded** (retire it; salvage only its keytool-signed HTTP helper for the downloader's auth).

**Decision (user-confirmed): mirror storj as faithfully as possible** — build a separate
`cmd/worker-updater` daemon AND reshape the version server to serve storj `AllowedVersions` +
computed rollout cursor. Sanctioned deviations only: (a) depin module/import paths and NodeID type
(`vo.UUID`), (b) keytool-signed HTTP auth on the download/poll requests (depin's transport), (c)
gin as the server framework. Everything else — file layout, decision precedence, verify/backup/swap/
restart flow, cursor math — ports 1:1 from `storj/cmd/storagenode-updater/`,
`storj/private/version/checker/`, and `storj/versioncontrol/`.

---

## 1. `pkg/version` — finish the port

- **Enable build info**: uncomment `pkg/version/info.go` `init()` → `Build = getInfoFromBuildInfo()`
  (function already present at `version.go:300`). Wire worker + updater builds with
  `-ldflags "-X aioz-depin/pkg/version.buildVersion=… -X …buildTimestamp=… -X …buildCommitHash=… -X …buildRelease=…"`.
- **Add process entries**: extend `Processes` (`version.go:60`) with `Worker Process json:"worker"` and
  `WorkerUpdater Process json:"worker-updater"` (keep storj fields for compatibility). These are the
  two binaries the daemon manages.

## 2. Version server — reshape `versioncontrol/` to storj's model

Port storj's `versioncontrol` rollout machinery onto depin's gin server (keep the existing
`GET /:file-name` binary/checksum download path — it becomes the `Version.URL` target).

- **Serve `AllowedVersions`**: add an endpoint (e.g. `GET /` or `/versions`) returning
  `version.AllowedVersions{Processes{Worker, WorkerUpdater, …}}` — populate the currently-dead
  `Server.versions` field (`api/api.go:22`) instead of leaving it nil. Per process serve
  `{Minimum, Suggested, Rollout{Seed,Cursor}}`; each `Version.URL` is a template with `{os}`/`{arch}`
  placeholders pointing at the download path.
- **Rollout config + cursor ramp**: port storj `versioncontrol` `RolloutConfig{Seed hex, Cursor 0–100,
  PreviousCursor}` + `ValidateRollouts` + `calculateRolloutCursor` (linear `PreviousCursor→Cursor`
  interp, rate-limited by the already-present `SafeRate`/`RegenInterval` in
  `versioncontrol/config/config.go`), emit the 32-byte cursor via `version.PercentageToCursorF`.
  Recompute on `RegenInterval`; `SafeRate<=0` → jump to full.
- Keep the existing build-tag download/serve as the artifact backend; the new layer only adds the
  storj-shaped **decision inputs** (minimum/suggested/rollout) the daemon needs.

## 3. Version checker client — port `private/version/checker/`

New `internal/version/checker/` (or `worker/version/checker/`):
- `client.go` — `Client.All(ctx) (version.AllowedVersions, error)`: GET the server, decode JSON;
  `Client.Process(ctx, name)` picks the process field by kebab→Pascal name (`worker-updater` →
  `WorkerUpdater`). Add depin's keytool-signed auth headers to the request (reuse the header layout
  from `worker/version/http_gateway.go`). Config `{ServerAddress, RequestTimeout}`.
- (Optional, storj §3/§8) `service.go`/`chore.go` — in-process notify-only version check
  (`IsAllowed`, `ErrOutdatedVersion`, `GetCursor`) for the worker peer. Lower priority; mark optional.

## 4. The updater daemon — new `cmd/worker-updater/` (1:1 with storagenode-updater)

Port each file, swapping only imports/NodeID:

| storj file | depin file | content |
|---|---|---|
| `cmd.go` | `cmd/worker-updater/cmd.go` | cobra root `worker-updater` + subcommands `run` / `restart-service` / `should-update`; `runCfg` flags (§ below); load identity, require non-zero nodeID |
| `loop.go` (+`loop_windows.go`) | `loop.go` | `loopFunc`: `checker.All(ctx)`; `update(... Processes.Worker)`; `update(... Processes.WorkerUpdater)`; failures log, never abort |
| `update.go` | `update.go` | `update()`: `binaryVersion` → `version.ShouldUpdateVersion(current, nodeID, ver)` → skip/last-failed/`downloadBinary`→verify version→`tryRunBinary(--help)`→`copyToStore`→backup→`restartAndCleanup`; `last-failed-update.<svc>.json` tracking |
| `binary.go` | `binary.go` | `binaryVersion` (run `<bin> version`, parse `Version: ` line), `downloadBinary`, `unpackBinary` |
| `path.go` | `path.go` | `parseDownloadURL` (`{os}`/`{arch}`→GOOS/GOARCH), `prependExtension` backup naming |
| `restart_util.go` | `restart_util.go` | `swapBinaries` rename dance (rollback on failure) |
| `restart_linux.go` | `restart_linux.go` | `swapBinaries`; standalone→done; updater-self→`exit=true`; else systemctl MainPID + SIGINT |
| `restart_windows.go`/`restart_bsd.go`/`restart.go` | same | platform variants (windows service swap; bsd swap-only) |
| `main.go` | `main.go` | entrypoint |

**`runCfg` flags** (mirror storj): `--binary-location` (default `aioznode`), `--updater-binary-location`,
`--binary-store-dir`, `--service-name` (default `aioznode`), `--restart-method` (`kill`\|`service`),
`--standalone`, `--identity-dir`, `--version.check-interval` (`15m`, min 1m clamp),
`--version.server-address`, `--version.request-timeout` (`1m`). Interval handling: `<=0` run once;
`0<iv<1m` clamp to 1m; else `sync2.NewCycle`.

**Backup naming** (storj): worker → `aioznode.old.<version>`; updater → `aioznode-updater.old`
(stable). **Restart**: Linux SIGINTs the worker's MainPID (supervisor relaunches); updater updating
itself returns `exit=true` and `os.Exit(1)`s so its supervisor relaunches it.

## 5. Make binaries report a parseable version

`binaryVersion` runs `<binary> version` and parses `Version: `. Ensure **both** the worker
(`cmd/worker`, root `aioznode`) and `cmd/worker-updater` expose a `version` subcommand printing
`version.Build.String()` (which includes `Version: vX.Y.Z`). Add the subcommand if absent.

---

## Files touched (representative)

- **New:** `cmd/worker-updater/` (the table above), `internal/version/checker/client.go`.
- **Edit:** `pkg/version/info.go` (enable init), `pkg/version/version.go` (Worker/WorkerUpdater
  processes), `versioncontrol/api/{api.go,handler.go}` + new rollout/cursor code +
  `versioncontrol/config/config.go` (RolloutConfig), worker/updater build ldflags + `version` subcmd.
- **Retire:** `worker/version/` in-process `AutoUpdate`/`SelfUpdate` (superseded by the external
  daemon); salvage its keytool auth-header helper for the checker/downloader.
- **Reuse (no change):** `pkg/version.ShouldUpdateVersion`/`isRolloutCandidate`/`PercentageToCursorF`,
  `internal/sync2.Cycle`, `pkg/identity` (nodeID `vo.UUID`), `cmd/worker/helpers.go:LoadPrivKey`
  (download auth), `worker/pkg/sysinfo`.

## Verification

1. `go build ./...`; `go test ./pkg/version/... ./internal/version/checker/... ./cmd/worker-updater/...`
   — port storj's `update_test.go`/`binary_test.go`/`cmd_test.go` (adapted) + keep `pkg/version` tests
   green (rollout cursor, `ShouldUpdateVersion` precedence).
2. **Real version**: build worker+updater with ldflags; `aioznode version` and `aioznode-updater
   version` print `Version: vX.Y.Z` (not `0.0.0`); `binaryVersion` parses it.
3. **`should-update` one-shot**: `worker-updater should-update worker` exits 0/1 matching
   `ShouldUpdateVersion` for the loaded nodeID + served cursor.
4. **End-to-end**: run the reshaped `cmd/versioncontrol` serving `AllowedVersions` (Suggested newer
   than the worker, cursor `ffff…ff`) + the binary at the templated URL. Run `worker-updater run
   --check-interval=…` → observe poll → decide → download → verify (`version`==expected, `--help`
   runs) → backup `aioznode.old.<v>` → swap → SIGINT worker PID → supervisor relaunches on new
   binary → backup removed.
5. **Staged rollout**: set a partial cursor; nodes with `HMAC(seed,nodeID) > cursor` log "hasn't made
   it to this node yet" and don't update; `ffff…ff` updates everyone; below-`Minimum` updates
   unconditionally (rollout bypassed).
6. **Failure safety**: serve a binary whose `--help` fails → recorded in
   `last-failed-update.worker.json`, binary deleted, no swap, worker keeps running; next tick skips
   the same version.

## Notes / risks

- **Restart assumes a supervisor** (systemd/docker) that relaunches on exit — that's storj's model.
  `--standalone` swaps files without restarting for unsupervised setups.
- **Download auth is a deviation**: storj does a plain GET; depin's server requires keytool-signed
  headers, so the daemon's HTTP client adds them. Binary integrity still relies on storj's
  run-verify (`version` + `--help`); optionally keep depin's `.checksum` verify as an extra step.
- **Two version worlds**: this reshapes the server onto storj's `AllowedVersions`+cursor model; the
  old build-tag filter stays only as the artifact-serving backend behind the templated `Version.URL`.
- `--service-name`/`--binary-location` default to `aioznode` (the worker binary); confirm the actual
  installed service/binary names before shipping.
