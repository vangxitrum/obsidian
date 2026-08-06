---
type: decision
tags: [depin, worker, autoupdate, versioncontrol, pkg-version]
created: 2026-07-08
updated: 2026-07-17
agent: main
---

Implemented the storj-faithful worker auto-update port per
`~/personal/tuan/Projects/depin/docs/worker-autoupdate.md` on branch `worker-autoupdate`
(rebased onto origin/main at 23a04d8). All 5 plan steps done + verified end-to-end
(should-update, full download→verify→backup→swap→restart cycle, failure-safety path all
exercised live against a real versioncontrol server + fake binary).

**New/changed:**
- `pkg/version`: enabled `init()` (Build populated from linker vars), added
  `Worker`/`WorkerUpdater` to `Processes`.
- `versioncontrol/`: added rollout cursor machinery (`config.RolloutConfig`,
  `ProcessesConfig`, `versioncontrol/api/rollout.go`'s `calculateRolloutCursor` ported from
  storj `versioncontrol/peer.go`), new `GET /versions` endpoint serving
  `version.AllowedVersions`, regen loop via `internal/sync2.Cycle`.
- `internal/version/checker/`: new client (`Client.All`/`Client.Process`/`Client.Download`),
  ported from storj `private/version/checker/client.go`, but every request is
  keytool-signed (depin's server requires auth on all routes, storj's doesn't) —
  this client is now also the download client for `cmd/worker-updater/binary.go`.
- `cmd/worker-updater/`: full daemon ported 1:1 from storj `cmd/storagenode-updater`
  (cmd.go/loop.go/update.go/binary.go/path.go/restart_{util,linux,bsd,windows}.go/main.go +
  windows variants). Uses `internal/sync2.Cycle` for the poll loop (user's explicit choice
  over the near-duplicate `pkg/common/sync2.Cycle`).
- `pkg/process/exec_conf.go`'s shared `cmdVersion` (auto-registered "version" subcommand on
  every cobra root via `ExecWithCustomOptions`) now prints `version.Build.String()` instead
  of a hardcoded "1.0.0" — this satisfies the "add version subcommand" plan step for
  **every** depin binary (worker, worker-updater, coord, keytool), not just the two named.
- Retired `worker/version/` (old in-process AutoUpdate/SelfUpdate); nothing else referenced it.
- Makefile.build: added `build-worker-updater` target — **must build via `./cmd/worker-updater`
  (package path), not `./cmd/worker-updater/*.go` (glob)** — glob bypasses Go's filename-based
  GOOS/GOARCH exclusion for `_windows`/`_linux`-suffixed files, so `*_windows.go` gets compiled
  on Linux and fails. `build-worker`'s existing `*.go` glob pattern works only because that
  dir has no platform-suffixed files.

**Bugs found+fixed (pre-existing, exposed by turning this feature on for the first time):**
1. `git tag v0.0.1` had to be added locally — repo had zero tags, so
   `git describe --dirty --tags --long` (used by `Makefile.build`'s `GIT_DESC` for
   `-X pkg/version.buildVersion=...`) produced `23a04d8-dirty`, not valid semver, and
   `getInfoFromBuildInfo()` **panics** on unparseable `buildVersion` once `init()` is enabled.
   This affects ALL binaries in the repo, not just worker-updater.
2. `pkg/version/semver.go` `SemVer.String()` had a leading-dot bug in prerelease formatting
   (`v0.0.1-.0-g23a04d8-dirty` instead of `v0.0.1-0-g23a04d8-dirty`) — broke the round-trip
   `binaryVersion()` needs (`<bin> version` → parse own `Version: ` line back via
   `NewSemVer`). Fixed by joining `sem.Pre` parts with `strings.Join(...,  ".")` instead of
   leading-concat. This was also a **pre-existing failing test**
   (`TestSemVer_String`) that nobody had noticed because nothing exercised the live
   round-trip before. Now passes.

**Repo-wide pre-existing breakage unrelated to this work** (confirmed via `git stash` diff,
present before this session): `coord/relay/status.go` embed pattern missing file,
`pkg/infectious` asm build failure, `uplinksdk`/`ers_schema`/`eestream` API mismatches,
`internal/vo` `TestPieceIDScanNullAndEmpty` failure, `worker/pieces` `trustpkg.Dialer` type
mismatch. None touched by this branch.

**E2E verification recipe** (useful for next time): `keytool create <svc> --difficulty 4`
generates a fast low-difficulty TLS identity; `keytool new --identity-dir <dir>/<svc>
--private-key-path <dir>/<svc>/priv_key.json ...` generates the cosmos key
worker-updater's HTTP auth needs (same dir as the TLS identity, since
`identity.Config.PrivateKeyPath` defaults to `$IDENTITYDIR/priv_key.json`). Build a fake
"updated" binary as a zip (`zip -j name.zip script.sh`) served from the versioncontrol
server's `path_binary` dir at `<Suggested.URL after {os}/{arch} substitution>`. See
[[hub-overview]] for repo layout.

---

## 2026-07-17 update - now MERGED to main + test coverage + dev spawn flow

Feature is **merged to main** (commits `95f4f9c` "implement worker version checking and
rollout", `508bfe2` "implement worker-updater daemon"). Repo has a real `v0.0.1` tag now,
so `git describe` yields valid semver (`v0.0.1-69-g3813b19`) - the build-panic bug no
longer bites on this checkout. Worktree: `depin-b971d9/1/depin`. Uncommitted per
no-auto-commit rule.

**Manual e2e re-verified (all 6 scenarios green)** against current main using the real
worker binary + a real `versioncontrol` server + a real keytool identity: full cycle
(swap to v0.0.2), rollout-gate blocks (cursor 0), below-minimum bypasses rollout,
version-mismatch rejects + keeps old (`invalid version downloaded: wants X got Y`),
bad-`--help` writes failure memo + keeps old, failure-memo short-circuits re-download,
updater self-update (standalone = swap-only, no os.Exit).

**Automated tests added (Phase B):**
- `cmd/worker-updater/update_test.go` (NEW, package main, 7 tests) - the real e2e:
  `httptest`-style `net.Listener` serving the **real** `versioncontrol/api` router (real
  auth middleware) + real `checker.Client` with an in-process `keytool.GenerateNewKey`
  cosmos key; fake worker binaries are `#!/bin/sh` scripts (echo `Version: vX.Y.Z` on
  `version`, exit code on `--help`) zipped via `archive/zip`; sets the package-level
  `nodeID` directly; calls `update(...standalone=true...)`. Download-count assertion
  proves memo short-circuit.
- `pkg/version/version_test.go` - filled the empty `TestShouldUpdate` stub (was
  `// TODO`): all 4 `ShouldUpdateVersion` branches + `ShouldUpdate` rollout boundaries +
  a cursor-monotonicity property test.
- `versioncontrol/api/rollout_test.go` (NEW) - `calculateRolloutCursor` ramp math +
  `generateAllowedVersions` seed-decode / 100%-all-0xFF.
- `internal/version/checker/client_test.go` (NEW) - `All`/`Process`(kebab->Pascal:
  worker-updater->WorkerUpdater)/`Download` signed round trip + 404 + transport error.
- Mutation-checked: flipping `isRolloutCandidate`'s `<=` to `>=` makes both the pure and
  the e2e rollout tests fail, so they actually guard the gate.

**One production refactor** (`versioncontrol/api/api.go`): extracted
`func (s *Server) Routes(v auth.Verifier) *gin.Engine` out of `StartServer` (which built
the gin router inline then blocked on `router.Run`) so tests can serve the real router.
No behavior change. No change to `update.go`/`binary.go`/`restart_*.go` - the storj port
stays faithful.

**Dev spawn flow (Phase C):** `dev/spawn-workers.sh {start|stop|restart|status}` runs each
worker under a **file-watch bash supervisor** + a `worker-updater run --standalone`. This
design is load-bearing: on Linux dev the updater NEVER restarts the worker itself
(`--standalone` swaps then returns before signalling; non-standalone needs `systemctl
show` for PID discovery and ROLLS BACK the swap on failure). So the supervisor polls the
binary hash and relaunches on change - keeping `cmd/worker-updater/` untouched. Per-worker
binary `bin/worker-<i>`; `--service-name worker-<i>`; updater points at
`VC_SERVER_ADDRESS` (default `http://127.0.0.1:10000`). Also: `dev/versioncontrol/config.yaml`
(committed template; NB versioncontrol has NO `setup` cmd - plain `LoadFromYAML`),
`build-versioncontrol` Makefile target (NEW), air configs `dev/worker{1,2}/.air.toml`
re-wired (`bin=/bin/true` so air doesn't compete with the supervisor; cmd calls
`spawn-workers.sh restart <i>`).

**Gotchas discovered this session:**
- **Served binary version must EXACTLY equal `suggested.version`** - `update.go:84` does
  `Compare != 0`. A GIT_DESC build reports `v0.0.2-0-g<hash>` which semver treats as a
  *prerelease of* v0.0.2 and sorts BELOW it -> mismatch. Build the served binary with
  `-X ...buildVersion=v0.0.2` (exact) for the version to match.
- **Semver prerelease trap for rollout tests:** current `v0.0.1-69-g...` sorts below
  `v0.0.1`, so `minimum: v0.0.1` hits the below-minimum branch and bypasses the cursor.
  Rollout-gating tests/scenarios MUST use `minimum: v0.0.0`.
- `parseDownloadURL` (`path.go:17`) only substitutes `{os}`/`{arch}`, no `{version}`.
- `sync2.Cycle.Run` fires the fn IMMEDIATELY on start (`cycle.go:105`, delayStart=false),
  so `worker-updater run` polls at once even with `--version.check-interval 1m` (floored
  at 1m; `<=0` = single-shot).
- `keytool create <svc> --difficulty 4 --identity-dir <dir>` writes `priv_key.json`
  directly (the older "separate `keytool key new` step" note is STALE).
- **`.gitignore` allowlist gotcha** (again, see [[depin-gitignore-allowlist-gotcha]]):
  `dev/spawn-workers.sh`, `dev/versioncontrol/config.yaml`, `binaries/.gitkeep` were all
  silently ignored by the top-level `*`; had to add explicit `!` allowlist lines.
- `pkill -f 'bin/versioncontrol'` KILLS YOUR OWN SHELL (its command line contains the
  string); kill by pid from `ps` instead, or put the pattern in a script file.

Full plan: local plan file
`~/.claude/plans/now-let-test-worker-parallel-metcalfe.md` (hermes was rate-limited
[HTTP 402 MONTHLY_REQUEST_COUNT] so it was NOT written to the vault Projects/depin/plans/).

## 2026-08-04 addendum - monorepo version identity

Storj uses one neutral `vMAJOR.MINOR.PATCH` Git tag as the source-release identity for
the whole monorepo. Satellite, storagenode, updater, uplink, and other artifacts built
from that tagged checkout normally embed the same semantic version plus commit/build
provenance; the tag is not called a satellite or storagenode version. Independent
component deployment happens in versioncontrol instead: every process has separate
minimum, suggested, download URL, rollout seed, and cursor, so a running satellite,
storagenode, and updater may be artifacts from different repository releases. Software
SemVer is also separate from Storj's monotonic node API-capability version.

Apply the same model to DePIN: keep neutral repo tags, name archives/images by component,
and control worker/updater (and coord if needed) independently in deployment policy.
DePIN already has distinct Worker and WorkerUpdater rollout entries. Current build sharp
edges: `git describe --tags --long` embeds `vX.Y.Z-0-gHASH` even exactly on a tag, and only
`release-coord` explicitly sets release=true; release builds should instead embed the
exact tag for every shipped artifact and verify version, commit, release, and dirty state.

## 2026-07-17 addendum - rollout-at-scale test + store-dir gotcha

`dev/rollout-scale-test.sh [N] [cursors...]` (NEW) - the "many machines on one host"
test for the rollout cursor. Spawns N distinct keytool identities (cached in
`tmp/rollout-scale/ids`, reused across cursor values so updated-sets NEST/monotonic),
one versioncontrol server, and N **single-shot** updaters (`--version.check-interval 0`,
each runs once + exits - far lighter than N daemons), then counts how many fake binaries
swapped to v0.0.2. Fake workers = shell scripts, no coord/db needed. Verified N=40 over
cursors 0/30/50/100 -> 0/40, 10/40, 16/40, 40/40 (tracks cursor; N drives precision,
binomial noise at small N). This is the only way to see the % gate - a single node only
ever sees cursor 0 or 100.

**GOTCHA: `--binary-store-dir` must be an EXISTING directory.** `copyToStore`
(`update.go:136`) does `os.OpenFile(storeDir/base, O_CREATE...)` which creates the FILE
but NOT the parent dir; a missing store dir => `open .../worker: no such file or
directory` at copyToStore:166 => the WHOLE update aborts (logged at loop.go:48, no swap,
leaves the `<bin>.<version>` download behind because unpackBinary uses O_EXCL). Callers
MUST `mkdir -p` the store dir first (spawn-workers.sh and rollout-scale-test.sh both do).
Set `--binary-store-dir ""` to skip copyToStore entirely. Faithful to storj (docker store
path always exists there), so NOT changed - just a caller requirement.
