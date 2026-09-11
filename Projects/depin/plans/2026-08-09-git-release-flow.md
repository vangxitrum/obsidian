---
type: plan
project: depin
tags: [depin, ci-cd, git-flow, release, branching]
created: 2026-08-09
agent: main
---

# Git Flow branching + pre-merge-only validation

> Supersedes the previous plan in this file (the docker-bake release system), which is
> built and shipped. Its outcome lives in `docs/RELEASE.md`.

## Deliverable order

**Phase 1, now: documentation only. No CI or code changes.**

1. `docs/BRANCHING.md` - new. The detailed git and release flow: the four-stage model,
   what each branch is for, a worked example of a normal release, all three hotfix
   cases, release branch lifetime, and how the versioncontrol rollout interacts with
   branching. This is the doc being asked for.
2. `docs/RELEASE.md` - updated to match: §2 points at the new doc instead of describing
   a flow that no longer applies, §4 cutting-the-release commands change to go through
   `release-X.Y`, and §11's "fix on `main` first" is corrected (it is wrong under this
   model and would send people to patch the one branch that will not run tests).
3. Archive this plan to the Obsidian vault at
   `/home/tuan/personal/tuan/Projects/depin/plans/2026-08-09-git-release-flow.md`,
   via the hermes CLI per the project convention, and refresh the link in
   `Projects/depin/INDEX.md`. Note hermes has recently been returning
   `HTTP 404: No active credentials for provider: kiro`; if it fails again the file
   gets written directly and that is reported rather than passed off as done.

**Phase 2, later and separately: the CI changes** described in "Changes" below (rules,
cache, duplicate pipelines, `check:backmerge`). Deliberately not bundled: the branching
model is a team decision that should be written down and agreed before pipeline
behaviour changes to depend on it, and the protection settings in Prerequisites have to
be in place first or the gate has a hole.

## Context

The question that started this: lint and test already pass on `develop`, so why run
them again when merging into `main`?

I first argued they were not redundant, because `main` had 3 commits `develop` did not.
**That was wrong.** Checking properly:

```
commits on main not in develop         : 3  - all three are MERGE commits
non-merge commits on main not in develop: 0
git diff main develop                  : IDENTICAL trees
```

Nothing has ever landed on `main` outside a `develop` merge, and every merge commit
carried a tree identical to the `develop` tree it merged. Re-running lint and test on
`main` re-validates a byte-identical tree. The intuition behind the question was right.

**Measured from `../storj`**, which is the project this repo is told to stay aligned
with (`CLAUDE.md`), and which faces the same staged-rollout problem:

```
v1.99.0-rc   cut at 9380fe8f2, then  1 commit      2024-02-28
v1.99.3      cut at 9380fe8f2, then 15 commits     2024-03-08
```

One release branch, cut from trunk once, patched for nine days with fixes only
(`storagenode/blobstore: fix disk space on windows`, `cmd/satellite: fix single part
mode calculation`, ...) while trunk kept moving. No release tag is on Storj's `main`.
Their pre-merge Jenkinsfile (317 lines) runs everything; their post-merge Jenkinsfile
(159 lines) runs build and publish and **no tests at all**.

That is the model adopted here, with `main` retained because it is useful to be able to
read "what is released" from a branch rather than only from tags.

Two unrelated inefficiencies get fixed in passing:

- **`main` always builds cold.** `.go-job` uses `key: go-mod-$CI_COMMIT_REF_SLUG`,
  scoping the cache per branch, so nothing reuses what `develop` just built.
- **Duplicate `develop` pipelines.** With a `develop -> main` MR open, a push to
  `develop` starts a branch pipeline *and* an MR pipeline for the same SHA, and the MR
  pipeline is a strict superset.

## The model

```
feature/* ──► develop ──► release-X.Y ──► main ──► tag vX.Y.Z
                  ▲            │
                  └────────────┘  backmerge fixes
   full validation   full validation   build + publish only
```

- `develop` is latest. Work that is merged but not shippable lives here.
- `release-X.Y` is cut at release time and frozen: fixes only, while `develop` moves on.
- `main` receives the release branch and carries the tag.
- Validation runs where changes are proposed and stabilised. `main` and tags only build
  and publish, because the tree was already validated on the branch it came from.

## Prerequisite, and it is the whole gate

Removing lint and test from `main` means **branch protection becomes the only thing
enforcing that validated code reaches `main`.** Before merging this change:

- **Settings -> Repository -> Protected branches**: `main` and `release-*`, merge by
  maintainers, **allowed to push: No one**.
- **Settings -> Merge requests -> Pipelines must succeed**: enabled.

Without both, a direct push to `main` ships untested code straight to `image:push`.
This is a settings change, so it cannot be verified from the repo and must be confirmed
in the UI. It is not optional.

## Changes

### 1. Validation stops running on `main` and tags

`.gitlab/ci/lint.yml` (`lint`) and `.gitlab/ci/test.yml` (`test`,
`test:compile-check`). All three carry identical rules today; replace with:

```yaml
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    # main and tags are only ever reached by merging an already-validated branch, so
    # the tree here was tested on develop or release-X.Y and is byte-identical. This
    # mirrors Storj, whose post-merge Jenkinsfile contains no tests. What makes it
    # safe is branch protection plus "pipelines must succeed", not this file.
    - if: $CI_COMMIT_BRANCH == "main" || $CI_COMMIT_TAG
      when: never
    - if: $CI_COMMIT_BRANCH
```

**`lint:release-tooling` keeps its current rules and still runs everywhere.** It takes
seconds and checks that build files are tracked and scripts executable, which matters
most at release time, exactly where the others are being switched off.

No changes needed to `build:*` / `image:push` / `publish:*`: they already gate on
`main`, `release-*` and tags.

### 2. Guard the backmerge, which is Git Flow's classic failure

A fix lands on `release-X.Y`, ships, and is never merged back to `develop`, so the next
release silently regresses it. Add to `.gitlab/ci/lint.yml`:

```yaml
check:backmerge:
  extends: .go-job
  stage: lint
  allow_failure: true          # informational: it is normal to be ahead mid-release
  script:
    - |
      missing="$(git log --oneline origin/develop..HEAD --no-merges)"
      if [ -n "$missing" ]; then
        echo "On this release branch but NOT yet in develop:"
        echo "$missing"
        echo
        echo "Merge this branch back into develop before closing the release,"
        echo "or these fixes disappear in the next one."
      fi
  rules:
    - if: $CI_COMMIT_BRANCH =~ /^release-/
```

`allow_failure: true` on purpose: being ahead of `develop` is the normal state during a
release, so this reports rather than blocks.

### 2b. Hotfixes: no new branch type, and one existing instruction is now wrong

`docs/RELEASE.md` §11 currently says "Fix on `main` first, then cherry-pick onto the
release branch". Under this model `main` is released code, not where work lives, so that
becomes **fix on `develop` first**. Leaving it as-is would send people to patch the one
branch that no longer runs tests.

Three cases, documented explicitly because only the middle one is unusual:

1. **During rollout, release branch still open.** Fix lands on `develop`, cherry-pick to
   `release-X.Y`, merge to `main`, tag `vX.Y.Z+1`, roll the versioncontrol cursor. This
   is Storj's `v1.99.1/.2/.3` and is routine, not an emergency path.
2. **Emergency on a closed release line.** Recreate the branch from the tag,
   `git checkout -b release-0.3 v0.0.3`, rather than inventing `hotfix/*`. Two reasons:
   it matches `release-*` so it inherits the full pipeline and the `build:*` /
   `image:push` rules for free, and a `hotfix/*` branch matches nothing in
   `workflow.rules` and would get **no pipeline at all** unless someone remembered to
   open an MR. Do not merge this into `main` when `main` already holds a newer release;
   tag off the release branch and cherry-pick to `develop` only if the bug survives
   there.
3. **True emergency, no time to route through `develop`.** Fix on `release-X.Y`, ship,
   backmerge immediately. `check:backmerge` (change 2) is what catches a forgotten one.

**No `hotfix/*` branch type is introduced.** If that ever changes, `workflow.rules` and
the `build:*` rules must be updated together, or such a branch runs nothing.

**Release branch lifetime**: keep `release-X.Y` until no worker is on that line and
`minimum.version` has passed it. The fleet runs several versions simultaneously, so a
line stays patchable long after the next one ships.

Note the safety property this preserves: `release-*` is a full-validation branch, so a
hotfix MR into it runs the entire suite. Only `main` and tags skip validation, and
neither is a place fixes are authored. The fast path cannot bypass the gate.

### 3. Split the cache so modules are shared and build output is seeded

`.gitlab/ci/lint.yml`, in `.go-job`:

```yaml
cache:
  # Modules are a pure function of go.sum and identical on every branch. Keying on the
  # file rather than the ref is what lets a fresh release-X.Y reuse develop's downloads.
  - key:
      files:
        - go.sum
    paths:
      - .gocache/mod
  # Build output stays per-branch but seeds from develop instead of starting cold,
  # which is what makes cutting a release branch cheap.
  - key: go-build-$CI_COMMIT_REF_SLUG
    fallback_keys:
      - go-build-develop
      - go-build-main
    paths:
      - .gocache/build
```

`GOMODCACHE` / `GOCACHE` already point at those directories.

`cache:fallback_keys` needs GitLab 15.6+; `cache:key:files` needs only 12.5 and carries
most of the benefit. The version was not readable without authentication, so it is
checked in Verification step 1. If older, drop `fallback_keys` and use a fixed
`go-build-shared` key.

### 4. Suppress the duplicate `develop` pipeline

`.gitlab-ci.yml`, `workflow.rules`, after the MR and tag rules:

```yaml
    # A push to develop while its MR is open is already tested by that MR's pipeline,
    # same SHA, superset of jobs. Scoped to develop deliberately: main's pipeline
    # publishes images and must never be suppressed.
    - if: $CI_COMMIT_BRANCH == "develop" && $CI_OPEN_MERGE_REQUESTS
      when: never
```

### 5. Document the model

`docs/RELEASE.md`:
- **§2 Branches and cadence**: replace `develop -> main` with the four-stage flow above,
  including that `release-X.Y` is frozen at cut and takes fixes only, and that it must
  be merged back into `develop`.
- **§4 Cutting the release**: the commands change. Cut `release-X.Y` from `develop`,
  stabilise, merge to `main`, then `tag-release.sh`. Today it tags straight off `main`.
- **§5 What CI does**: note in the stage table that lint and test do not run on `main`
  or tags, and why.
- **§11 Hotfixes**: rewrite for the three cases in change 2b. The existing "fix on
  `main` first" instruction is actively wrong under this model and must go. State that
  there is no `hotfix/*` branch type and why, and give the release-branch lifetime rule.
  Cross-reference §10's versioncontrol levers, which are unchanged and still correct.
- **Appendix: one-time setup**: the two protection settings, as required configuration.

## Verification

1. **Version check** before relying on `fallback_keys`: Help -> Help, or
   `curl -s -H "PRIVATE-TOKEN: <pat>" https://gitlab.internal/api/v4/version`.
2. **Validate with GitLab's own CI linter**, which catches an unsupported cache keyword
   that a YAML parse will not:
   `curl -s -H "PRIVATE-TOKEN: <pat>" --data-urlencode "content=$(cat .gitlab-ci.yml)"
   "https://gitlab.internal/api/v4/projects/<id>/ci/lint"` expecting `"valid":true`.
3. **Confirm the protection settings in the UI.** After change 1 they are the only gate,
   and nothing in the repo can prove they are on.
4. **Walk the full flow once, on throwaway refs.** Cut `release-0.1` from `develop`:
   confirm lint, test, test:compile-check, build:binaries, build:package, image:push all
   run, and `check:backmerge` reports clean. Add a fix commit to it: confirm
   `check:backmerge` now lists that commit. Merge to `main`: confirm the pipeline runs
   build and publish jobs **only**, with lint and test absent from the pipeline graph
   rather than skipped-and-green. Tag it: confirm the release jobs run and lint/test are
   still absent. Delete the throwaway tag and branch afterwards.
5. **Cache.** In the `release-0.1` pipeline, look for `Successfully extracted cache`
   naming a `go-build-develop` fallback, and confirm `test` does not re-download the
   module graph. Compare `test` duration against the last `develop` run.
6. **Duplicate removal.** With a `develop -> main` MR open, push to `develop`: exactly
   one new pipeline, of type `merge request`. Then close the MR and push again: a branch
   pipeline must run. That second half is the regression this rule could plausibly
   cause, so check it explicitly.
7. **A hotfix still gets validated.** Recreate a release branch from an existing tag
   (`git checkout -b release-0.0 v0.0.2`), open an MR into it, and confirm the MR
   pipeline runs the full lint and test set. This is the property the whole plan rests
   on: the emergency path must not be the one that skips the gate.

## Non-goals

- Merge trains / merged results pipelines, which would test the merge result before it
  lands. Premium; `gitlab.internal` is Community Edition (confirmed from
  `/-/manifest.json`). If it is ever upgraded, they supersede this arrangement.
- The tree-equality guard on `main` proposed earlier and verified against real history.
  Dropped in favour of the simpler Storj shape plus branch protection.
- Touching `-race` in `test`. Largest single cost, but load-bearing for a system this
  concurrent.
- A post-merge migration test, the equivalent of Storj's `CRDB Run Rolling Upgrade
  Test`. This is the valuable thing to put in the time this frees up, since it tests
  something the pre-merge pipeline structurally cannot, but it is its own work.

## Open item, unrelated

`docs/RELEASE.md` §5 still carries a "KNOWN RED as of 2026-08-07" block for
`TestRepairRebuildsLostPiece`, which now passes. It measured 4 failures in 8 runs
against the pre-fix tree, so re-run it several times against the current tree before
rewriting or deleting that block.
