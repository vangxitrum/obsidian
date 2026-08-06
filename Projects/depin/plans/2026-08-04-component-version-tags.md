---
type: plan
project: depin
created: 2026-08-04
status: proposed
tags: [depin, versioning, component-tags, build, storj-port]
---

# Component version tags (source tag from repo tag, per-component tag per binary)

## Context

Every depin binary is stamped with one repo-wide version string. `Makefile.build:15`:

```make
GIT_DESC := $(shell git describe --always --dirty --tags --long)
```

`GIT_DESC` feeds `-X aioz-depin/pkg/version.buildVersion` for **all** targets (`build-coord`, `build-worker`, `build-edgeserver`, ...). Consequences today:

- Every binary reports `v0.0.1-180-g702ee23`, so `coord version`, `worker version` and the value coord persists in `workers.version` (`coord/db/migrations/000033_worker_version.up.sql`) say nothing about which component actually changed.
- The repo has exactly one tag (`v0.0.1`, 180 commits behind HEAD), so the version is effectively "the repo, some time ago".
- `--always` is a latent startup crash: on a clone with no reachable tag it degrades to a bare hash, `NewSemVer` fails, and `pkg/version/version.go:356` `panic(err)` fires inside `init()`. A shallow CI clone reproduces this.
- Docker images are pinned to static `:latest` (`Makefile.build:38,41,45`), so a deployed image cannot be traced to a build.

Storj solves this in `storj/scripts/bake.sh`: on an exact repo tag every component gets that tag; otherwise each component's version is derived from the last commit touching *that component's* directories. Same checkout, different version per component. That is the model adopted here.

**Outcome:** each buildable binary carries its own version derived from its own git history, plus a new `Source:` field carrying the repo-level tag, stamped identically into every binary. No new git tags to create or push.

## Decisions (confirmed)

1. **Tag source** - derived from git history (storj `bake.sh` model). Exact repo tag wins; otherwise per-component commit-derived. No per-component git tags.
2. **Scope** - all buildable binaries: `coord`, `worker`, `worker-updater`, `edgeserver`, `uplink`, `keytool`, `versioncontrol`, `relay-ping`.
3. **versioncontrol server** - unchanged. It keeps serving only `worker` + `worker-updater`. This is a build/stamping change only; `pkg/version.Processes`, `versioncontrol/config`, `versioncontrol/api/rollout.go` are not touched.

## Version string shape (deviation from bake.sh - important)

`bake.sh` emits a year-major string (`v2026.08.<epoch>-<hash>`). Adopting that shape verbatim would break the worker rollout: `worker-updater` compares its running version against the versioncontrol `suggested` version with `SemVer.Compare` (`pkg/version/version.go` `ShouldUpdateVersion`), and `v2026.8.x` sorts **above** any real release tag like `v0.0.2`, so a worker running a dev build would permanently report "Version is up to date" and never accept a release. Storj is immune only because its release tags are `v1.x`.

Derived versions therefore use a shape that always sorts **below** any release tag, with the ordering carried in a numeric prerelease identifier:

```
exact repo tag         ->  v0.1.0                              (all components identical)
untagged / dev build   ->  v0.0.0-dev.<unix-ts>.g<short-hash>  (per component)
```

Ordering, verified against `blang/semver/v4` semantics:

```
v0.0.0-dev.1753901234.g16663b7  <  v0.0.0-dev.1754300000.g702ee23  <  v0.0.0  <  v0.0.1  <  v0.1.0
```

Two constraints this shape satisfies, both verified in this repo:

- **No `+build` metadata.** `pkg/version/semver.go:49` builds the metadata string as `build = build + "." + val`, producing a stray leading dot (`v1.2.3+.linux.amd64`) on re-render. Prerelease join (`:43`, `strings.Join(parts, ".")`) is correct, so all extra data goes in the prerelease.
- **Hash prefixed with `g`.** A bare short hash that happens to be all digits with a leading zero is rejected by blang as a numeric prerelease identifier. `g702ee23` can never be all-digits. (`ParseTolerant` at `semver.go:240` strips leading zeros from major/minor/patch, so month-style values would have parsed, but the prerelease rule still applies.)

## Files to change

### 1. `scripts/component-version.sh` (new)

Single source of truth. Prints one line: the build version for one component.

```bash
#!/usr/bin/env bash
set -euo pipefail
comp="${1:?usage: component-version.sh <component>}"

# CI / explicit override.
if [ -n "${BUILD_VERSION:-}" ]; then echo "$BUILD_VERSION"; exit 0; fi

case "$comp" in
  coord)           dirs="cmd/coord coord" ;;
  worker)          dirs="cmd/worker worker" ;;
  worker-updater)  dirs="cmd/worker-updater internal/version" ;;
  edgeserver)      dirs="cmd/edgeserver edgeserver" ;;
  uplink)          dirs="cmd/uplink" ;;
  keytool)         dirs="cmd/keytool certificate worker/pkg/keytool" ;;
  versioncontrol)  dirs="cmd/versioncontrol versioncontrol" ;;
  relay-ping)      dirs="dev/relay-ping" ;;
  *) echo "unknown component: $comp" >&2; exit 1 ;;
esac
# Shared code every component links against.
dirs="$dirs pkg internal go.mod go.sum"

# On an exact repo tag, the source tag IS every component's version.
if tag=$(git describe --tags --exact-match --match 'v[0-9]*.[0-9]*.[0-9]*' 2>/dev/null); then
  echo "$tag"; exit 0
fi

read -r ts hash <<<"$(git log -1 --format='%ct %h' -- $dirs 2>/dev/null)"
echo "v0.0.0-dev.${ts:-0}.g${hash:-unknown}"
```

Component dir sets were derived from actual imports (verified: `cmd/keytool` imports `aioz-depin/certificate` and `aioz-depin/worker/pkg/keytool`; `internal/keytoolcmd` from an older memory note no longer exists). `pkg` + `internal` are shared by everything, matching storj's `./shared ./private ./go.mod ./go.sum` in both dir sets: a change there correctly bumps every component. Narrowing this is a later tuning knob, not a correctness fix.

### 2. `scripts/source-version.sh` (new)

The repo-level source tag, identical in every binary. Display-only, never semver-parsed, so `--always --dirty` is safe here:

```bash
#!/usr/bin/env bash
set -euo pipefail
if [ -n "${SOURCE_VERSION:-}" ]; then echo "$SOURCE_VERSION"; exit 0; fi
git describe --tags --always --dirty --match 'v[0-9]*.[0-9]*.[0-9]*' 2>/dev/null || echo "unknown"
```

### 3. `pkg/version/version.go` - new `Source` field

- Add `buildSource string` to the linker-flag var block (`:29-34`), documented as "repo-level source tag; display only, not semver".
- Add ``Source string `json:"source,omitempty"` `` to `Info` (`:41-50`).
- In `getInfoFromBuildInfo()` (`:296`), assign `rv.Source = buildSource` near the other linker fallbacks. No Go build-info equivalent exists, so it is a plain assignment with no precedence rules and no parse - it can never panic.
- `Info.String()` (`:140`): add `out += fmt.Sprintln("Source:", info.Source)` when non-empty.
  **Hard constraint:** `cmd/worker-updater/binary.go:31-42` `parseVersion` scans for `strings.HasPrefix(line, "Version: ")`. `"Source: "` does not collide. Do not name it anything beginning with `Version: `.
- `Info.Log()` (`:166`): add `zap.String("Source", info.Source)`.

Note on scope: only `buildVersion` (and the new `buildSource`) is per-component. `buildCommitHash` and `buildTimestamp` ldflags are **already dead** in a normal git build - `version.go:339` and `:347` only apply when Go's own VCS stamping left them empty, and `vcs.revision`/`vcs.time` are always populated from HEAD when `.git` is present. That is correct behaviour (the binary really was built from HEAD), and the per-component commit is preserved inside the version string's `g<hash>` suffix anyway. Do not add `-buildvcs=false` to chase this.

### 4. `Makefile.build` - per-component ldflags

Replace the `GIT_DESC` block (`:13-24`) with a component-parameterised macro:

```make
PKG            := aioz-depin/pkg/version
TIMESTAMP      := $(shell date -u +%s)
GIT_HASH       := $(shell git rev-parse --short HEAD)
SOURCE_VERSION := $(shell ./scripts/source-version.sh)
BUILD_RELEASE  ?= false

# $(call ldflags,<component>) - recursively expanded, so the git call runs
# once per build target actually invoked.
define ldflags
-s \
 -X $(PKG).buildVersion=$(shell ./scripts/component-version.sh $(1)) \
 -X $(PKG).buildSource=$(SOURCE_VERSION) \
 -X $(PKG).buildCommitHash=$(GIT_HASH) \
 -X $(PKG).buildTimestamp=$(TIMESTAMP) \
 -X $(PKG).buildRelease=$(BUILD_RELEASE)
endef
```

Then each build target passes its own component name, e.g. `build-coord` (`:55-61`):

```make
	@CGO_ENABLED=1 GOOS=$(GOOS) GOARCH=$(GOARCH) go build -tags '$(GO_BUILD_TAGS)' \
	  -ldflags '$(call ldflags,coord)' -o $(BIN_DIR)/$(COORDINATOR_BIN) ./cmd/coord/*.go
```

Apply the same pattern to `build-worker` (`:182`), `build-worker-updater` (`:190`), `build-versioncontrol` (`:200`), `build-keytool` (`:67`), `build-keytool-signer` (`:78`, component `keytool`), `build-relay-ping` (`:87`), `build-edgeserver` (`:228`), `build-uplink` (`:225`). Update each target's `> Version:` echo line to print its own component version instead of `$(GIT_DESC)`.

Related changes in the same file:

- **`release-*` for every component.** Today only `release-coord` exists (`:63-65`). Replace the pattern with `BUILD_RELEASE` so any target can be built in release mode (`BUILD_RELEASE=true make build-worker`), and keep `release-<comp>` aliases for the components that ship. Keep this **opt-in, never auto-derived from "HEAD is tagged"** - `version.Build.Release` silently flips config defaults via `internal/cfgstruct/cfgstruct.go:83,93,103` (`release:` vs `devDefault:` struct tags), so it must stay an explicit act.
- **Versioned docker tags, additive.** `dev/fleet/*.sh`, `docker-compose.yml` and `dev/fleet/package-worker.sh` all reference `:latest`. Keep `:latest` as-is and add a second tag, e.g. in `build-coord-image` (`:101-107`): `docker build -f Dockerfile.coord -t $(COORD_IMAGE) -t coord:$(shell ./scripts/component-version.sh coord) .`. Same for `build-edgeserver-image` (`:116`) and `build-fleet-image` (`:131`, tagged with the `worker` component version). Zero breakage, traceable artifacts.
- **New `version-info` target** printing the source tag plus every component version - the manual verification surface and a debugging aid.
- **Optional cleanup:** `build-hub`/`run-hub`/`setup-hub` (`:215-222`) and `run-workermanager` (`:246`) reference `cmd/hub` and `cmd/workermanager`, neither of which exists. Delete them rather than inventing dir sets for phantom components.

### 5. `dev/fleet/fleet.sh` - keep the operator-chosen version, fix the provenance

`fleet.sh rollout <ver>` (`:209-230`) hand-rolls its own ldflags and hardcodes `buildCommitHash=fleet`. The explicit `<ver>` argument is the point of the command and stays. Change only:

- add `-X aioz-depin/pkg/version.buildSource="$(./scripts/source-version.sh)"`,
- replace the literal `fleet` with `$(git rev-parse --short HEAD)`,

so a rolled-out worker binary is traceable to a commit and a source tag. Everything else (zip packaging, `write_vc_config`, `docker restart fleet-versioncontrol`) is untouched.

### 6. Docs

`MAINTAINERS.md` is unadapted Storj boilerplate (references a non-existent `branch.py`, `master`, storj release-branch numbering, four literal `TODO`s). Add a **Versioning** section stating: the repo tag is the source tag; on an exact tag every component ships that tag; otherwise each component's version derives from its own git history; `make version-info` shows the current values. Do not attempt a wider MAINTAINERS.md rewrite here.

## Tests

- **`pkg/version/version_test.go`** - add cases asserting:
  - `NewSemVer("v0.0.0-dev.1754300000.g702ee23")` parses;
  - ordering `dev(older) < dev(newer) < v0.0.0 < v0.0.1` via `SemVer.Compare` (this is the regression guard for the rollout hazard above);
  - `Info.String()` on an `Info` with `Source` set contains exactly one line with prefix `Version: ` and one `Source: ` line.
- **`cmd/worker-updater/update_test.go`** (existing file) - add a `parseVersion` case fed the full new `Info.String()` output including the `Source:` line, asserting it still resolves the component version. This is the contract the updater depends on.

## Verification

1. `make version-info` - source tag plus 8 component versions; all derived versions must be distinct where the components' histories differ.
2. `make build-coord build-worker && ./bin/coord version && ./bin/worker version` - both print a `Source:` line with the same repo tag and a `Version:` line that differs between the two binaries (worker and coord have different last-touching commits on this branch).
3. Touch only `worker/`, commit, rebuild both: worker's version advances, coord's does not. This is the whole feature in one check.
4. Tagged case: in a scratch clone, `git tag v9.9.9 && make version-info` - every component prints `v9.9.9`.
5. Shallow-clone regression: `git clone --depth=1 file://$PWD /tmp/shallow && make -C /tmp/shallow build-worker && /tmp/shallow/bin/worker version` - must print a version rather than panicking in `init()` (this is the `--always` bug being fixed).
6. `./bin/worker version | grep -c '^Version: '` must be exactly `1` - the updater's parse contract.
7. `go test ./pkg/version/... ./cmd/worker-updater/...`.
8. Fleet e2e: `make build-fleet-image && dev/fleet/fleet.sh rollout v0.0.6 100 0` then `dev/fleet/fleet.sh status` - workers still report the operator-chosen `v0.0.6`, and `docker exec ... worker version` now also shows a real commit hash and `Source:`.

## Out of scope (flagged, not fixed here)

- **Duplicate migration `000033`.** `coord/db/migrations/000033_worker_version.*` and `000033_worker_tags.*` collided at the `feat/worker-tags` merge that is HEAD. `golang-migrate` rejects duplicate version numbers, so migrations will fail to load. Pre-existing and unrelated to this change, but it will bite before this can be exercised end-to-end.
- **CI.** This branch has no CI. `origin/create_cicd`'s `.gitlab-ci.yml` builds only `cmd/worker` with `-ldflags="-s -w"` - i.e. **no version stamping at all** - and derives `APP_VERSION` from `git describe --tags --abbrev=0` purely for the artifact filename. When that branch merges, its build jobs should call `make build-worker` (optionally with `BUILD_VERSION=`/`BUILD_RELEASE=true`) so CI artifacts carry the same stamps as local builds.
- `versioncontrol` still serves only `worker` and `worker-updater`, per decision 3.

## Related

- [[worker-autoupdate]] - the rollout consumer whose `SemVer.Compare` drives the version-shape constraint above
- [[2026-07-22-worker-tags-port]] - unrelated despite the name: node metadata labels, not versions
- [[hub-overview]]
