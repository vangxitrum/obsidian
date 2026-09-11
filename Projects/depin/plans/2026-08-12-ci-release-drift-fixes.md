---
type: plan
project: depin
tags: [depin, ci-cd, release, gitlab-ci, docker-tags]
created: 2026-08-12
agent: main
---

# Fix three CI drifts from the documented release model

> Follows [[2026-08-09-git-release-flow]], the plan that established this
> branching model. This one fixes where the CI config drifted from it.

## Context

The question that prompted this was "when do we create a tag?" The branching and CI model
you described is already designed, documented, and implemented in this repo - the YAML
matches the docs. So this plan is **not** a redesign. It fixes three places where the
config has quietly drifted from the documented model, and leaves the model itself alone.

### The model, and where tagging sits

```
feature/* ──► develop ──► release-X.Y ──► main ──► tag vX.Y.Z
                  ▲            │
                  └────────────┘  backmerge
```

**The tag is set by hand on `main`, after the release-branch MR has merged**
(`docs/RELEASE.md:104-108`). It is not what triggers the merge; it is what you stamp on the
merge result. Nothing in CI creates a git tag, and nothing should.

A release builds four times, and only the last one counts:

| # | action | pipeline | `BUILD_VERSION` stamped |
|---|---|---|---|
| 1 | `git checkout -b release-0.0 && git push -u` | `release-0.0` branch | `v0.0.0-dev.<ts>.g<hash>` |
| 2 | cherry-pick fixes onto `release-0.0` | `release-0.0`, per push | `v0.0.0-dev.*` |
| 3 | open MR `release-0.0` → `main` | merge request | `v0.0.0-dev.*` |
| 4 | merge the MR | `main` | `v0.0.0-dev.*` |
| 5 | **`tag-release.sh v0.0.3` && `git push origin v0.0.3`** | **tag** | **`v0.0.3`, `buildRelease=true`** |

Builds 1-4 are validation throwaways: the tag does not exist yet, so
`scripts/component-version.sh:35`'s `git describe --exact-match` finds nothing and
`BUILD_VERSION` falls through to the dev form. Only the tag pipeline runs `build:package`,
`publish:sign`, `publish:release`, `publish:versioncontrol-config` and `verify`
(`.gitlab/ci/release.yml`, all four gated on `- if: $CI_COMMIT_TAG` alone).

The version is never read from a `VERSION` file - there isn't one. `CI_COMMIT_TAG` is
injected as `BUILD_VERSION` in the release jobs (`.gitlab/ci/release.yml:33-34`), and the
`BUILD_VERSION` env override at `component-version.sh:5-9` is the escape hatch if a build
ever needs a version before its tag exists.

### How the tag reaches the artifacts, given it is created last

The tag pipeline does **not** promote `main`'s build - it rebuilds the same commit. That is
what makes tagging-after-merge work:

```bash
git checkout main && git pull          # main now holds the merged release
./scripts/release/tag-release.sh v0.0.3   # creates the tag locally, does NOT push
git push origin v0.0.3                 # this is what starts the release
```

The push matches the workflow rule `- if: $CI_COMMIT_TAG` (`.gitlab-ci.yml:156`), starting a
second pipeline on that same commit. There `build:binaries` re-runs (`build.yml:47`) and
`scripts/bake.sh:33,36` resolves differently than it did on `main`:

```bash
GIT_TAG="$(git describe --tags --exact-match --match 'v[0-9]*.[0-9]*.[0-9]*' ...)"   # -> v0.0.3
BUILD_VERSION="${BUILD_VERSION:-${GIT_TAG:-v0.0.0-dev.${BUILD_TIMESTAMP}.g${BUILD_COMMIT}}}"
```

Same tree, different ldflags: `buildVersion=v0.0.3`, `buildRelease=true` (`bake.sh:43-46`,
`release.Dockerfile:164-169`). Identical source, different binary, different image digest.

This is also why `.gitlab-ci.yml:42-43` pins `GIT_DEPTH: 0` and `GIT_STRATEGY: clone`:
`git describe` needs full history and tags, and a shallow clone would silently stamp a dev
version onto a real release. Do not touch those two variables.

`main`'s own build stays (decided) - it proves the merge commit builds even when tagging is
deferred, and publishes an immutable `:<short-sha>` image for deploying `main` between
releases. It is never a release artifact.

### Decisions taken

- **Tagging stays manual on `main`.** No CI job will create or push a git tag.
- **Merge requests keep build and test.** No change to MR behaviour.
- **`main` keeps its build.** It validates the merge commit and publishes `:<short-sha>`;
  it is never a release artifact.
- **`:latest` becomes a release pointer.** The tag pipeline pushes both `:vX.Y.Z` and
  `:latest`; `main` stops moving it. Release candidates do not move it - see Task 2.
- **Three drifts get fixed** - Tasks 1, 2, 3 below.

---

## Target behaviour

| ref                | lint                             | test / coverage / compile-check | build  | image  | publish      |
| ------------------ | -------------------------------- | ------------------------------- | ------ | ------ | ------------ |
| merge request      | yes                              | yes                             | yes    | yes    | no           |
| `develop`          | yes                              | yes                             | **no** | **no** | no           |
| `develop`, MR open | no pipeline at all (unchanged)   |                                 |        |        |              |
| `release-X.Y`      | yes + `check:backmerge`          | yes                             | yes    | yes    | no           |
| `main`             | no (`lint:release-tooling` only) | **no**                          | yes    | yes    | no           |
| tag `vX.Y.Z`       | no (`lint:release-tooling` only) | **no**                          | yes    | yes    | full release |

Bold cells are what changes. Every one of them already reads this way in
`docs/BRANCHING.md:254-267` - the config just doesn't do it.

### Resulting image tags

`TAG` is immutable per pipeline, `LATEST_TAG` is the moving pointer; both from
`scripts/bake.sh:48-70`. Task 2 changes one cell.

| ref | `TAG` | `LATEST_TAG` |
|---|---|---|
| exact semver tag `v0.0.3` | `v0.0.3` | **`latest`** (was empty) |
| exact rc tag `v0.1.0-rc1` | `v0.1.0-rc1` | empty |
| `main` | `<short-sha>` | **empty** (was `latest`) |
| `release-1.2` | `<short-sha>-release-1.2` | `release-1.2-latest` |
| any other branch | `<short-sha>-<branch>` | empty |

Task 2 swaps which ref owns `:latest`: the **tag** pipeline moves it, `main` no longer does.
`:latest` therefore means "the newest released version" rather than "the tip of `main`".

---

## Tasks

### 1. Fix the `coverage` rules drift - `.gitlab/ci/test.yml`

`coverage` (`test.yml:60-87`) extends `.postgres-job` and defines **no `rules:` of its
own**, so it inherits `.postgres-job`'s (`test.yml:33-40`), which carry no `main`/tag
exclusion. A full Postgres suite therefore runs on every merge to `main` and every release
tag, contradicting the documented "no test on main or tags". It is `allow_failure: true`,
which is why nobody noticed.

Add an explicit `rules:` to `coverage`, matching what `test` and `test:compile-check`
already carry at `test.yml:125-136`:

```yaml
  rules:
    - if: $SKIP_VALIDATION == "true"
      when: never
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    # Skipped on main and tags: the tree there already passed on develop or
    # release-X.Y and is byte-identical. Reasoning in .gitlab/ci/lint.yml and
    # docs/BRANCHING.md section 9.
    - if: $CI_COMMIT_BRANCH == "main" || $CI_COMMIT_TAG
      when: never
    - if: $CI_COMMIT_BRANCH
```

Leave `.postgres-job`'s own rules alone - `coverage` was its only consumer lacking an
override.

### 2. Move `:latest` from `main` to the tag pipeline - `scripts/bake.sh`

Today `:latest` tracks `main`, which is wrong in two ways. `bake.sh:56-58` sets
`LATEST_TAG="${LATEST_TAG:-latest}"` on `main`, so every merge republishes `:latest`; and
per the release timeline above, `main` builds at step 4 **before** the tag exists at step 5,
so `BUILD_VERSION` is `v0.0.0-dev.<ts>.g<hash>`. `:latest` therefore holds binaries that
self-report a dev version. Meanwhile the tag pipeline - the one that actually produces
`v0.0.3`, `buildRelease=true` artifacts - publishes only `:v0.0.3` and moves nothing.

Swap the two branches of the tag-scheme `if` (`bake.sh:55-58`):

```bash
if [ -n "${TAG:-}" ]; then
  : # explicit override wins
elif [ -n "$GIT_TAG" ]; then
  TAG="$GIT_TAG"
  # A release moves :latest. Release candidates do not - see below.
  case "$GIT_TAG" in
  *-rc*) LATEST_TAG="${LATEST_TAG:-}" ;;
  *)     LATEST_TAG="${LATEST_TAG:-latest}" ;;
  esac
elif [ "$GIT_BRANCH_NAME" = "main" ]; then
  TAG="$BUILD_COMMIT"
  LATEST_TAG="${LATEST_TAG:-}"
else
  ...unchanged...
fi
```

Keep the `${LATEST_TAG:-...}` form throughout so an explicit `LATEST_TAG=` from the
environment still wins. Update the scheme comment at `bake.sh:48-54`, which documents the
old `main`-owns-latest behaviour and is the first thing anyone reads.

**The `-rc` guard is an assumption, flag it for confirmation.** `tag-release.sh:30` accepts
`vX.Y.Z-rcN`, and bake's match glob `v[0-9]*.[0-9]*.[0-9]*` (`bake.sh:33`) **does** match
`v0.1.0-rc1` - the trailing `[0-9]*` swallows `0-rc1`. So without the `case` above, tagging
a release candidate would move `:latest` onto it. The plan assumes that is unwanted. If rc
tags should move `:latest`, drop the `case` and use the plain
`LATEST_TAG="${LATEST_TAG:-latest}"`.

**Consequence to state in the docs.** `:latest` now moves at *tag time*, which is **before**
the staged versioncontrol rollout (`docs/RELEASE.md:253-300`, cursor 5 → 25 → 100 with
soaks). Operators on the registry channel therefore get a new release as soon as it is
tagged, with no soak in front of them - whereas `docs/RELEASE.md:393-395` currently promises
`latest` moves "only after the rollout has completed and soaked". The two update channels
are already documented as independent (`RELEASE.md:255-257`); this makes the registry
channel strictly faster than the versioncontrol one. That is a real behaviour change for
anyone pulling `:latest`, and §8 must be rewritten rather than left contradicting the code.

**Check before merging.** Grep `dev/fleet/` compose files and `docs/RELEASE.md:215-220` for
`:latest`, and record the current registry state of `:latest` at `10.0.0.106:5000` so the
change is reversible.

`release-*` keeps its own `release-X.Y-latest` pointer - scoped to a release line, not
`:latest`, unaffected.

### 3. Fix the `develop` build asymmetry - `.gitlab/ci/build.yml`

`build:binaries-macos` (`build.yml:103-114`) and `image:push` (`build.yml:201-210`) each
end with a bare `- if: $CI_COMMIT_BRANCH` fallback. `build:binaries` (`build.yml:46-57`)
has no such fallback. Under the workflow rules (`.gitlab-ci.yml:153-167`) only `develop`
can ever reach that fallback - feature branches produce no pipeline at all. So `develop`
builds **macOS binaries and pushes an image with no Linux binaries beside them**, which is
both unintentional and contrary to `docs/BRANCHING.md:258` ("`develop`: build no,
publish no").

Remove the trailing `- if: $CI_COMMIT_BRANCH` from both jobs. Keep every other rule,
including the MR rules - MR behaviour is unchanged. Resulting rules for both:

```yaml
  rules:
    # (build:binaries-macos keeps its $DEPIN_MACOS_ENABLED guard first)
    - if: $CI_COMMIT_TAG
    # Skipped for docs-only or refactor work: SKIP_BUILD in .gitlab-ci.yml.
    # Below the tag rule on purpose, so a release always builds.
    - if: $SKIP_BUILD == "true"
      when: never
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_COMMIT_BRANCH =~ /^release-/
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

The `$CI_COMMIT_TAG`-first, `$SKIP_BUILD`-below ordering is load-bearing and must survive
the edit - a release that skipped its own build would publish signatures and a
versioncontrol pointer for binaries that do not exist (`.gitlab-ci.yml:120-123`).

The comment above `image:push` (`build.yml:179-194`) argues the image is "the deliverable
on every pipeline, so it is pushed on every pipeline." That stays true for MRs, `release-*`,
`main` and tags; it stops being true for `develop`. Adjust the wording rather than leaving
it to contradict the rules.

**Check before merging:** confirm nothing deploys from a `develop` image tag. If something
does, that is an argument for keeping the fallback on `image:push` only, and the docs
should change instead.

### 4. Add `tag-release.sh` to the tooling allowlist - `.gitlab/ci/lint.yml`

`scripts/release/tag-release.sh` is missing from the tracked-files list in
`lint:release-tooling` (`lint.yml:167-174`), unlike every other `scripts/release/*.sh`.
This repo has an allowlist `.gitignore`, and a dropped release script is exactly the
failure mode that job exists to catch. Since tagging is the one manual release step, the
script being present in a fresh clone matters. Add it.

### 5. Update the docs to match

Only where the config now disagrees:

- `docs/BRANCHING.md:254-267` - the `develop` row's build/publish cells become true after
  Task 3; no edit needed if they already say "no", but verify cell by cell.
- `docs/RELEASE.md:123-129` - same check for the `develop` row.
- `docs/RELEASE.md:393-405` (§8 "Promoting to latest") - after Task 2 the tag pipeline
  moves `:latest` automatically, so the manual `imagetools create` promote is no longer the
  mechanism. Rewrite the section: `:latest` now means "newest released tag", it moves at
  tag time rather than after soak, and the `imagetools` command survives only as the
  **rollback** lever for pointing `:latest` back at an older release.
- `docs/BRANCHING.md:26-36` (the "Status" section) claims the CI behaviour in §9 is "in
  place". After these tasks that becomes true; before them it overstates. Leave it, but
  re-read it once the tasks land.

---

## Explicitly out of scope

- **Merge request behaviour.** MRs keep `lint`, `lint:release-tooling`, `test`,
  `test:compile-check`, `coverage`, `build:binaries`, `image:push`. Unchanged.
- Any CI job that creates or pushes a git tag. Tagging stays manual.
- Deriving a version from a branch name or `VERSION` file. `git describe` +
  `$CI_COMMIT_TAG` remains the sole source (`scripts/component-version.sh:35-38`).
- The four-builds-per-release cost (~1.1GB of binaries each, per `build.yml:41-45`).
  Real, but it is the price of the documented model, not a drift from it.
- `.gitlab/releases/<tag>.md` does not exist in the repo, so `publish:release`'s
  `--description` (`release.yml:85`) is a dangling reference. The `release-notes` skill
  writes it at release time; noted, not fixed here.
- The three separate ldflags implementations (`release.Dockerfile:164-169`,
  `scripts/release/build-native.sh:318-322`, `Makefile.build:24-30`) have drifted slightly
  (`-s -w` vs `-s`). Unrelated to pipeline rules.

---

## Verification

No `glab`, `gitlab-ci-local`, or `yq` on this machine, so verification is static parsing
plus one real pipeline run.

**1. YAML parses, and the effective rules are what we think.** Write a throwaway script in
the scratchpad that loads `.gitlab-ci.yml` and each `.gitlab/ci/*.yml` with Python's
`yaml`, then prints every job's effective `rules:` **with `extends:` resolved** - so
`coverage` inheriting `.postgres-job` is visible rather than assumed. That inheritance is
precisely the Task 1 bug, so the check must model it or it proves nothing.

**2. Simulate every ref.** For the six rows in the target table (merge request, `develop`,
`develop` with an open MR, `release-0.0`, `main`, tag `v0.0.3`), evaluate each job's rules
top-to-bottom with the matching `CI_*` variables set and emit the job list. Diff against
the target table. Repeat with `SKIP_BUILD` and `SKIP_VALIDATION` each set to `"true"` - the
tag-rule-above-skip-rule ordering is the part most easily broken by an edit to Task 3.

**3. Exercise the tag scheme without pushing.** `scripts/bake.sh:48-70` derives `TAG` and
`LATEST_TAG` from `CI_COMMIT_REF_NAME` plus git state. Drive it for `main`, `release-0.0`, a
feature branch, the existing `v0.0.2` tag, and a scratch `v0.1.0-rc1` tag, asserting the
five rows of the tag table: `latest` **only** for the plain semver tag, empty everywhere
else including `main` and the rc. The rc case is the one most likely to be wrong, since it
depends on the `-rc` glob interaction at `bake.sh:33` - create the rc tag in a throwaway
clone under the scratchpad, never in the real repo. Use a sourced copy of the script if it
has no dry-run flag; do not let it invoke a real bake or push.

**4. One real run.** Push the branch and open an MR. Expected: the MR pipeline is
**identical to before this change** - `lint`, `lint:release-tooling`, `test`,
`test:compile-check`, `coverage`, `build:binaries`, `image:push`. Any difference means
Task 3 removed more than the `develop` fallback.

**5. Confirm the two fixes where they actually show.** Neither is observable on an MR
pipeline:
- After the MR merges to `develop`: `coverage` still runs, and `image:push` /
  `build:binaries-macos` no longer do.
- The `coverage`-on-`main` fix and the `main`-half of the `:latest` change only show on a
  merge to `main`. Verify on the next real release: `main`'s pipeline should publish
  `:<short-sha>`, run no `coverage`, and leave `:latest` pointing where it was.
- The tag-half of the `:latest` change shows only on the tag pipeline, which then publishes
  `:v0.0.3` **and** moves `:latest` onto it. Confirm both refs resolve to the same digest
  afterwards.

Do not verify by tagging a throwaway release: the tag pipeline signs and publishes to the
FTP release store and the package registry.

---

## Note on what this plan cannot enforce

`docs/BRANCHING.md:279-289` records that skipping lint and test on `main` is only safe
because `main` and `release-*` are protected with **allowed to push: no one**, and
"Pipelines must succeed" is enabled. Nothing in the repository can verify that. Since
Task 1 removes the last test job that was still incidentally running on `main`, that
setting is now the only gate. Confirm it in the GitLab UI.
