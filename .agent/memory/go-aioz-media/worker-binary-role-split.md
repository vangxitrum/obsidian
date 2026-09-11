# worker-binary-role-split

The worker is built from three `cmd/` packages that share all their code and
differ only by a `role.Role` value: `cmd/aiozworker` (combined, artifact
`aioz-depin-cli-*`, root command still named `aioznode`), `cmd/aiozstorage`
(`aioz-depin-storage-*`) and `cmd/aioztranscode` (`aioz-depin-transcode-*`).
Implemented 2026-09-10 on branch `feat/split-binanry`.

`worker/role/role.go` holds the three presets. `worker/rootcmd` builds the
command tree for a role and is the entry point of all three `main.go` files;
`storage get|set` only exists when `role.Storage`, hidden `version binaries`
only when `role.Transcode`.

Things that were non-obvious and are easy to get wrong again:

- **The size win comes only from the embed, not from Go code.** ffmpeg/ffprobe
  now live in `transcode/embedbin` (`//go:embed binaries/*`). The storage
  binary simply does not import that package, so the blobs are not linked.
  `worker/task-manager` still contains the transcode steps in every binary -
  that is deliberate and costs almost nothing. Guard: `go list -deps
  ./cmd/aiozstorage | grep embedbin` must be empty.
- **`transcode/embedbin/binaries/PLACEHOLDER` is committed on purpose.**
  Before the split, a local `go build` of the worker failed outright with
  `pattern binaries/*: no matching files found` unless CI had mirrored ffmpeg
  in. The placeholder makes the embed pattern match, so local builds work and
  a missing ffmpeg degrades to a runtime log line instead.
- **The viper env prefix used to be `path.Base(os.Executable())`.**
  `server.InterceptConfigsPreRunHandler` now takes an explicit `envPrefix`
  from the role, because renaming the executable silently changed which
  `<PREFIX>_*` env vars were read.
- `worker/rest.StartCommand` had a `registerRoutesFn` parameter that no caller
  ever used and `RegisterRoutes` ignored. It was removed; route selection is
  now `registerSharedRoutes` + `registerStorageRoutes` / `registerTranscodeRoutes`
  driven by the role. Guarded by `worker/rest/routes_role_test.go`.
- **No hub change was needed.** The hub picks transcoders in
  `hub/contract/controller/service.go:2726` from online workers filtered by
  `WorkerTranscodeCap.Ok` rows, which are only written as a side effect of a
  *challenged* transcode task. A storage-only node never earns one, and
  `/worker/assignTranscoding` 404s anyway. The residual case is a node that ran
  the combined binary, earned a cap, then got redeployed storage-only - the cap
  row survives 7 days. Fixing that needs a role advertised in the free-form
  `UpdateReq.metadata` JSON (`worker/rest/root.go`, today only `home_path`)
  plus a filter at `service.go:2744`. No proto change required.

Release cadence is independent only at the CI level: one shared
`types/version.Version`, but `.gitlab-ci.yml` has per-role jobs keyed on
`release/worker/*`, `release/storage/*`, `release/transcode/*`, and
`buildworker.sh <role>` / `create_release_zips.sh -a <artifact>` build a subset.

Both binaries default to `~/.aiozworker` and port 1317, so running two on one
host needs explicit `--home` and `--laddr` (the `singleprocess.lock` is
per-home). This was a deliberate choice to keep existing operator setups working.

Related: [[worker-process-states]], [[worker-ui-telemetry-gotchas]]
