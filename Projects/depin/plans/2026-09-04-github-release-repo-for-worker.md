---
title: 'Public GitHub release repo for the DePIN worker'
date: 2026-09-04
project: depin
status: implemented
---

# Public GitHub release repo for the DePIN worker

## Context

`AIOZNetwork/aioz-depin-cli` is a **distribution-only** repo: its whole tree is `LICENSE`
(Apache 2.0) + `README.md`. Every byte of payload ships as GitHub Release assets, built by a
private GitLab pipeline that pushes them over. `AIOZNetwork/aioz-ai-node-wrapper` is the same
shape, so this is an established org convention rather than a one-off.

The DePIN **worker** has no such front door. Operators get it out-of-band as a ~215MB
`dist/depin-worker-fleet.zip` Docker bundle (`dev/fleet/package-worker.sh`), and auto-update
pulls binaries from a self-hosted nginx box over plain HTTP
(`http://10.0.0.67:8821/worker-v0.0.11.zip`), with release artifacts shuttled between CI jobs over
plain FTP (`scripts/release/artifact-store.sh`). There is no public, verifiable download point.

The good news from recon: **the depin release pipeline is already complete and already emits
exactly the right artifacts.** `.gitlab/ci/build.yml` `build:package` (tag-only) runs
`check-release-binaries.sh` then `compress-binaries.sh`, producing
`release/<tag>/<binary>_<goos>_<goarch>.zip` plus `sha256sums`; `publish:sign` adds syft SBOMs and
a cosign `sha256sums.sig`. Today `publish:release` uploads that set to the **GitLab generic package
registry**. We are adding a second destination, not building a pipeline.

Decisions taken with the user:
- Source of truth: the **`depin` monorepo** (`git@gitlab.internal:aioz-depin/depin.git`).
- Publishing: **GitLab CI pushes to GitHub** on protected tags.
- Assets: **pipeline-native zips only** - no repackaging, no tar.gz.
- GitHub Releases **becomes the auto-update download source**.
- **Enable macOS** in the tag pipeline so darwin ships from day one.
- Prototype against `tuantq-aioz/new-depin`; the real org repo is created later by a human
  (the `gh` credentials here are `vangxitrum`, with no AIOZNetwork membership).

### One correction to an earlier claim

I said pointing versioncontrol at GitHub would let us fill `VersionConfig.Checksum` and enable
`--version.require-checksum`. That is **not** achievable in this change. `Version.URL` is a single
templated string (`.../worker_{os}_{arch}.zip`, substituted at poll time by
`cmd/worker-updater/path.go:17-21`) while `Version.Checksum` is a single scalar - one config entry
maps to N platform zips with N different digests. Making `require-checksum` usable needs a schema
change across `pkg/version/version.go`, `versioncontrol/config/config.go`,
`versioncontrol/api/rollout.go` and `cmd/worker-updater`. Out of scope here, listed as follow-up.
This change still upgrades the transport from plain HTTP on one box to HTTPS on GitHub's CDN, and
publishes a signed `sha256sums` operators can verify by hand.

## Part A - the public repo (`tuantq-aioz/new-depin` now, `AIOZNetwork/aioz-depin-worker` later)

Tree stays exactly two files, matching both sibling repos.

**`LICENSE`** - Apache 2.0, copied from `AIOZNetwork/aioz-depin-cli`.

**`README.md`** - modeled section-for-section on the `aioz-depin-cli` README:
`What is it` → `Requirements` → `Getting started` (per-OS download/extract/verify) → `Usage`.
Differences forced by the worker:

- A release is **per-binary**, not one bundle. An operator needs three assets per platform:
  `worker_<goos>_<goarch>.zip`, `worker-updater_<goos>_<goarch>.zip`,
  `keytool_<goos>_<goarch>.zip`. The README must say so plainly and show all three downloads.
- Checksum file is `sha256sums` (lowercase, no extension, `./`-prefixed entries - see
  `scripts/release/compress-binaries.sh:81`), **not** the `SHA256SUMS` that `aioz-depin-cli` uses.
  Do not rename it: `verify-release.sh` and the cosign signature are bound to that exact name.
  Add a verification snippet (`sha256sum -c sha256sums`, and `cosign verify-blob`).
- Command reference derives from `cmd/worker/command.go` - root command `aioznode`, subcommands
  `setup`, `run`, `api`, `config`, `storage`, plus `balance` / `withdraw <address>` from
  `cmd/worker/withdraw.go`. Identity minting is `keytool`, not a worker subcommand
  (`cmd/keytool/cmd_identity.go`, `cmd_keys.go`). Read `cmd/worker/entrypoint` for the flags the
  fleet actually passes (`--version.server-address`, coord trust source, difficulty).
- Carry over the cli README's **wallet-secret warning block** verbatim in spirit - same risk here.
- Keep the same "auto-update is experimental / the binary must be writable" note; the worker's
  updater swaps the executable in place (`cmd/worker-updater/update.go:107-133`).

No CI, no workflows, no source in this repo.

## Part B - depin monorepo changes (branch off `develop`)

### B1. `scripts/release/publish-github.sh` (new)

Sibling to the existing `scripts/release/*.sh`, same `set -euo pipefail` + `--flag` arg style as
`update-versioncontrol.sh`.

- Inputs: `--version <tag>`, `--repo <owner/name>`, `--dir release/<tag>` (default),
  `--notes-file .gitlab/releases/<tag>.md`; token from `$GITHUB_TOKEN`.
- Talks to `api.github.com` with `curl` (the `.release-tools` image is alpine + `apk add curl`;
  do not add a `gh` dependency for this).
- Creates the release if absent, otherwise reuses it; uploads `*.zip`, `sha256sums`,
  `sha256sums.sig`, `*.spdx.json`. **Idempotent**: delete an existing asset of the same name
  before uploading, so a retried job converges instead of erroring on duplicates.
- Marks the release as prerelease when the tag matches `-rc`, mirroring the rc guard the repo
  already applies elsewhere.
- Refuses to run if `sha256sums.sig` is missing - that is the signal that `publish:sign` did not
  complete, and an unsigned public release is worse than no release.

### B2. `.gitlab/ci/release.yml` - new `publish:github` job

Slots between `publish:sign` and `publish:versioncontrol-config`.

```yaml
publish:github:
  stage: publish
  image: $DOCKER_IMAGE
  needs:
    - job: publish:sign
      artifacts: false
  script:
    - apk add --no-cache bash curl
    - ./scripts/release/artifact-store.sh pull "${CI_COMMIT_TAG}/signed" "release/${CI_COMMIT_TAG}"
    - ./scripts/release/publish-github.sh --version "$CI_COMMIT_TAG" --repo "$DEPIN_GITHUB_RELEASE_REPO"
  rules:
    - if: $DEPIN_GITHUB_RELEASE_REPO == null
      when: never
    - if: $CI_COMMIT_TAG
```

Gating on `$DEPIN_GITHUB_RELEASE_REPO` means the job is inert until someone sets it, so this can
merge before the org repo exists. Note the file's own header rule: every job here needs a secret
and is tag-only, and `$GITHUB_TOKEN` must be added as a **protected + masked** CI variable.

### B3. Point auto-update at GitHub

`publish:versioncontrol-config` already reads `${DEPIN_RELEASE_BASE_URL:-<gitlab package url>}`
(`.gitlab/ci/release.yml:92-110`). So this is a **CI variable, not a code change**:

```
DEPIN_RELEASE_BASE_URL = https://github.com/<org>/<repo>/releases/download/$CI_COMMIT_TAG
```

`update-versioncontrol.sh` then emits `.../worker_{os}_{arch}.zip`, which is exactly the asset
name `compress-binaries.sh` produced, and GitHub asset names carry no path separators, so the
template resolves cleanly.

Two ordering fixes go with it:
- Add `needs: [publish:github]` to `publish:versioncontrol-config`, otherwise the generated config
  can name URLs before the assets exist.
- The worker-updater sends keytool-signed auth headers on the download (`client.go:185-230`);
  GitHub ignores unknown headers, so this works unchanged - same as the nginx box does today
  (`dev/fleet/nginx.conf:11-13`).

### B4. Enable macOS

Pure configuration: set `DEPIN_MACOS_ENABLED=true` as a project CI variable. `build:binaries-macos`
(`.gitlab/ci/build.yml:131-177`) is already written, `parallel: matrix` over `arm64`/`amd64`,
pinned by runner tag `darwin-$MAC_ARCH`, and `build:package` already pulls the `darwin-*` prefixes
with `optional: true`. Prerequisite is operational, not code: both mac runners registered with
those tags, Xcode CLT, Go matching `go.mod`, and gitlab.internal's cert in the system keychain.
`check-release-binaries.sh` is what decides whether the set is complete enough to ship.

### B5. Docs

`docs/RELEASE.md` §6 "Where the artifacts land" (line 209) and §7 (the two update channels, line
266) both describe the GitLab-registry-only world. Update them to name GitHub Releases as the
public channel and the auto-update base URL, and record the required CI variables.

## Files touched

| File | Change |
|---|---|
| `scripts/release/publish-github.sh` | new |
| `.gitlab/ci/release.yml` | add `publish:github`; add `needs` to `publish:versioncontrol-config` |
| `docs/RELEASE.md` | §6, §7, CI-variable table |
| `<public repo>/LICENSE`, `<public repo>/README.md` | new repo content |

Nothing in `release.docker-bake.hcl`, `compress-binaries.sh`, `sign-artifacts.sh`, or any Go
package changes - the artifacts are already right.

## Verification

1. **Script hygiene** - `bash -n scripts/release/publish-github.sh` and `shellcheck` it, matching
   how the other release scripts are kept.
2. **Real artifacts, locally** - `make release/binaries/build` then `make release/binaries/compress`
   on this machine produces `release/<ver>/worker_linux_amd64.zip` etc. plus `sha256sums`. (A
   populated `release/v0.0.0-dev.1785904513.g030e81a/` tree already exists to test against.)
3. **End-to-end publish against the sandbox** - run `publish-github.sh --repo tuantq-aioz/new-depin
   --version v0.0.1-test` with a `GITHUB_TOKEN`, then from a clean dir:
   - `curl -fLO https://github.com/tuantq-aioz/new-depin/releases/download/v0.0.1-test/worker_linux_amd64.zip`
   - `sha256sum -c sha256sums` passes
   - re-run the script unchanged and confirm it converges rather than erroring (idempotency).
4. **Templated URL resolves** - run
   `scripts/release/update-versioncontrol.sh --version v0.0.1-test --base-url https://github.com/tuantq-aioz/new-depin/releases/download/v0.0.1-test --cursor 100 --safe-rate 48`,
   then `curl -fLI` the `{os}/{arch}`-substituted URL for each shipped platform.
5. **Auto-update actually swaps** - point the `dev/fleet` stack's versioncontrol at that generated
   config (`fleet.sh` / `dev/fleet/versioncontrol.config.yaml`), start a worker one version behind,
   and confirm `worker-updater` downloads from GitHub, passes the single-entry zip check
   (`cmd/worker-updater/binary.go` `unpackBinary`), re-runs `worker version` to match, and restarts.
   This is the test that matters - it exercises the whole chain the operators depend on.
6. **CI dry run** - push a `v*-rc` tag on a throwaway branch with `DEPIN_GITHUB_RELEASE_REPO`
   pointing at the sandbox repo, and confirm the release lands marked prerelease.

## Follow-ups (not in this change)

- **Per-platform checksums** so `--version.require-checksum` can be turned on. Needs
  `Version.Checksum` to become a per-`{os}_{arch}` map across `pkg/version`, `versioncontrol/config`,
  `versioncontrol/api/rollout.go` and `cmd/worker-updater`. Until then every auto-update installs
  unverified bytes (`cmd/worker-updater/checksum.go:378-401` warns and proceeds).
- **Retire the FTP artifact store** once GitHub is the durable destination
  (`scripts/release/artifact-store.sh`).
- `docs/RELEASE.md:315-319` records a live known gap: fleet containers started with an empty
  `FLEET_CONTROL_HOST`, so their `VC_SERVER_ADDRESS` does not resolve and auto-update has been
  inert since launch. Pointing at GitHub does not fix that; those workers need re-provisioning.
- An operator convenience bundle (one archive per platform holding all three binaries) if the
  three-download flow proves awkward in practice.
