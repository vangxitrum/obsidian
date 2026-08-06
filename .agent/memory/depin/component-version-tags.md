# Component version tags (per-binary version + repo Source tag)

Status: implemented 2026-08-05 on branch `feat/comopent-tags`, worktree
`~/.treehouse/depin-b971d9/3/depin`. **Uncommitted.**
Plan: `Projects/depin/plans/2026-08-04-component-version-tags.md` (executed verbatim).

## What shipped

Storj `bake.sh` model, adapted. Every buildable binary now carries its own version
derived from its own git history, plus a repo-level `Source:` tag identical in all binaries.

- `scripts/component-version.sh <comp>` - per-component version. Exact repo tag wins;
  otherwise `v0.0.0-dev.<unix-ts>.g<short-hash>`. Honours `BUILD_VERSION` override.
- `scripts/source-version.sh` - repo tag via `git describe --always --dirty`. Display only,
  never semver-parsed. Honours `SOURCE_VERSION` override.
- `pkg/version`: new `buildSource` ldflag + `Info.Source` field, rendered as a `Source:` line
  in `String()` and a field in `Log()`.
- `Makefile.build`: `$(call ldflags,<comp>)` macro replaces the single repo-wide `GIT_DESC`;
  `BUILD_RELEASE` replaces the `LDFLAGS_RELEASE`/`GO_LDFLAGS` pattern; `release-*` aliases for
  coord/worker/worker-updater/edgeserver/versioncontrol/keytool; additive versioned docker tags
  alongside `:latest`; new `make version-info`; deleted phantom `build-hub`/`run-hub`/`setup-hub`/
  `run-workermanager` targets (`cmd/hub` and `cmd/workermanager` do not exist).
- `dev/fleet/fleet.sh rollout`: adds `buildSource`, and replaces the literal
  `buildCommitHash=fleet` with the real short HEAD.
- `MAINTAINERS.md`: new Versioning section.

Version shape deliberately sorts BELOW any release tag
(`v0.0.0-dev.<older> < v0.0.0-dev.<newer> < v0.0.0 < v0.0.1`) so a dev build cannot make
worker-updater report "up to date" forever. Guarded by a regression test.
All extra data goes in the prerelease, never `+build`, because `SemVer.String()` renders
build metadata with a stray leading dot.

## Traps found during execution (none were in the plan)

1. **`Makefile.migration:5` is a bare `export`.** Export-everything forces GNU make to expand
   every recursive variable when constructing any recipe's environment, so
   `define ldflags` ran `component-version.sh` with an empty `$(1)` on every target.
   Fix: `unexport ldflags` after the `endef`. Applies to any future `define` holding `$(shell ...)`.
2. **Allowlist `.gitignore`** needed `!scripts/*.sh`; without it the scripts never commit and a
   fresh clone builds with an empty version. The allowlist also rejects `.txt`, which silently
   broke a verification step until switched to `.md`.
3. **Plan's claim that `buildCommitHash`/`buildTimestamp` ldflags are dead is false here** -
   file-list builds (`./cmd/coord/*.go`) disable Go's `-buildvcs`, so the ldflags are live.

## Verification

Steps 1-7 pass. Per-component divergence, the exact-tag case, and the shallow-clone
no-panic regression were proved in a throwaway git repo (nothing committed to the branch).
Note at a merge-commit HEAD all components report the *same* version, because every dir set
includes shared `pkg internal go.mod go.sum` - divergence only appears after a commit touching
one component's own dirs.

Fleet e2e ran local-half only: `fleet.sh rollout v0.0.6` produced a binary stamped
`Version: v0.0.6` / `Source: v0.0.1-180-g702ee23-dirty` / real commit hash. The 50 live
`fleet-worker-*` containers were deliberately NOT touched - see below.

## Incidental findings, unrelated to this change

- The 50 running `fleet-worker-*` containers come from `work/depin-workspace/depin/dev/fleet`
  (compose project `fleet`, created 2026-07-24), point at coordinator `68.183.189.51:7777`,
  and were started in workers-only mode with `FLEET_CONTROL_HOST` empty. They therefore fell
  back to `VC_SERVER_ADDRESS=http://versioncontrol:10000`, which resolves to nothing on this
  host. **Their auto-update has been inert since launch** - binary timestamp equals creation date.
- `uplink/` package dir does not exist, so `build-uplink` was already broken before this change.
- `../go-sdk` symlink absent, so `build-edgeserver` cannot compile here.
- `fleet.sh rollout` rewrites the tracked `dev/fleet/versioncontrol.config.yaml`. Reverted after
  verification; worth knowing it dirties the tree.
