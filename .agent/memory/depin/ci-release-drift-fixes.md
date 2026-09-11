---
type: decision
tags: [depin, ci-cd, gitlab-ci, release, docker-tags]
created: 2026-08-12
agent: main
---

# CI/release-model drift fixes shipped

Executed [[2026-08-12-ci-release-drift-fixes]] (vault plan) verbatim on branch
`feat/cicd`, uncommitted. Fixes 3 places `.gitlab/ci/*.yml` + `scripts/bake.sh` had
drifted from `docs/BRANCHING.md`/`docs/RELEASE.md`'s already-documented model - not a
redesign. See [[docker-bake-release-pipeline]] for the release pipeline this builds on.

## What changed

1. `.gitlab/ci/test.yml` - `coverage` job had no own `rules:`, so it inherited
   `.postgres-job`'s (no main/tag exclusion) and ran a full allow_failure Postgres
   suite on every `main` merge and release tag. Added explicit rules matching
   `test`/`test:compile-check`.
2. `scripts/bake.sh` - swapped which ref owns `:latest`. Was: `main` moves it on
   every merge (stamping dev-version binaries as `latest`, since `main` builds
   *before* the tag exists in the 4-build release timeline). Now: the **tag**
   pipeline moves it, gated `*-rc*) empty ;; *) latest ;;` so release candidates
   never move it. `main` now publishes `<short-sha>` only, no moving tag.
3. `.gitlab/ci/build.yml` - `build:binaries-macos` and `image:push` each had a
   trailing bare `- if: $CI_COMMIT_BRANCH` fallback that only `develop` could ever
   reach (workflow rules block feature branches from pipelines at all). Removed
   from both; `develop` now builds/publishes nothing, matching docs. `build:binaries`
   already lacked the fallback (asymmetry was the bug).
4. `.gitlab/ci/lint.yml` - added missing `scripts/release/tag-release.sh` to
   `lint:release-tooling`'s tracked-files allowlist check.
5. `docs/RELEASE.md` §8 "Promoting to latest" rewritten: `:latest` now moves
   automatically at tag-push time (not after the staged versioncontrol rollout
   soaks - the two channels are independent, registry is now strictly faster).
   `docker buildx imagetools create` demoted from "the promote mechanism" to
   "the rollback lever." `docs/BRANCHING.md` needed no edits (already correct).

## Plan-gap found mid-execution, user-approved fix

Plan's Task 5 doc-edit list didn't include `docs/RELEASE.md` §6 "Tag scheme" table
(right above §8), which the bake.sh swap also falsified (`main | latest` and
`tag | none` were now backwards, plus a "latest is not moved by tagging" sentence
directly contradicting Task 2). Stopped and asked per execute-plan-verbatim skill;
user said fix it too, same rationale as §8. Table + sentence corrected.

## Second plan-gap found, user-approved fix

`.gitlab-ci.yml`'s top-of-file header comment ("main / develop -> the above (also
moves latest)") was also stale on both halves - the `develop` half predated this
plan (matches the drift Task 3 fixes), the `latest` half newly falsified by Task 2.
Not in plan's Task 5 scope (scoped to `docs/BRANCHING.md`/`docs/RELEASE.md` only).
Flagged, user said fix it too; comment rewritten to the actual per-ref pipeline
shape.

## Verification done (plan's steps 1-3, all green)

- Python/pyyaml script resolving `extends:` confirmed `coverage`'s effective rules
  now match `test`/`test:compile-check` exactly.
- Full ref simulation (merge request / develop / develop-with-open-MR / release-0.0 /
  main / tag) matched the plan's target table exactly, including with
  `SKIP_BUILD=true` (tag-rule-above-skip-rule ordering survived intact) and
  `SKIP_VALIDATION=true`.
- bake.sh tag-scheme dry run in a throwaway scratch clone (never the real repo,
  never pushed) for main/release-0.0/feature-branch/existing-v0.0.2-tag/scratch-
  v0.1.0-rc1-tag: all 5 rows matched exactly. Gotcha hit twice: `git clone` only
  picks up committed history, not working-tree edits, so the dry-run script must be
  rebuilt from the live edited `scripts/bake.sh`, not from the clone's stale copy -
  and any throwaway commit/tag made for one scenario silently confounds a later
  scenario built on the same branch tip (`git describe --exact-match` matches by
  commit, not by branch identity) unless scenarios are ordered so no two share a
  commit.

Plan's steps 4-5 (real MR pipeline run, post-merge confirmation on `main`/a real
release tag) are explicitly deferred to actual CI - not run here.

Registry check during Task 2: `10.0.0.106:5000` had no `:latest` tag for any
component yet (all `MANIFEST_UNKNOWN`), so the change starts from a clean slate.

## Follow-up shipped same session: MR pipeline is now build-test only

Same branch, same session. Brainstormed + planned + implemented: `build:binaries`,
`build:binaries-macos`, `image:push` all branch their `script:` on
`$CI_PIPELINE_SOURCE == "merge_request_event"` - on an MR they still build the exact
same bake/Dockerfile matrix (so a broken cross-compile still fails the MR) but skip
`publish-build.sh`/`prune-store.sh` (FTP) and `registry-login.sh`/`--push`
(registry). `release-*`/`main`/tag pipelines are byte-identical to before. No new
CI variables, no new jobs - reuses the existing `.buildx` template's persistent
BuildKit `--keep-state` cache volume so the post-merge real publish build is a
cache hit. Plan file: `~/.claude/plans/scalable-herding-coral.md`.

Plan-premise bug caught before implementing: the approved plan told me to add a
doc note "near the four-builds-per-release timeline table" in `docs/RELEASE.md`,
assuming that table lived there. It doesn't - that table only exists in the
vault plan `2026-08-12-ci-release-drift-fixes.md` (this doc's own §4 has the same
steps in prose, no table, no cost callout). User chose to skip that doc item
rather than invent a location.

Verified by extracting the actual YAML `script:` blocks via pyyaml, stubbing every
external script call as a bash function, `bash -n` syntax-checking, then running
under both `CI_PIPELINE_SOURCE=merge_request_event` and `=push`: MR path never
calls `publish-build.sh`/`prune-store.sh`/`registry-login.sh` and bakes images
without `--push`; non-MR path is call-for-call identical to pre-change. Real
cache-hit speedup on a post-merge build is unverifiable statically - deferred to
an actual pipeline.
