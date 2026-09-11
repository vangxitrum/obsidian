---
type: decision
tags: [release, ci, github, worker, versioncontrol]
created: 2026-09-04
agent: main
---

Public GitHub Releases added as a **second** publish destination for the worker,
modeled on `AIOZNetwork/aioz-depin-cli`. Branch `feat/github-release-publish` off
`develop`, uncommitted at time of writing.

## What the org convention actually is

`aioz-depin-cli` and `aioz-ai-node-wrapper` are **distribution-only** repos: the whole
git tree is `LICENSE` (Apache 2.0) + `README.md`, and every byte of payload is a GitHub
Release asset pushed there by a private GitLab pipeline. Naming is
`<product>-<os>-<arch>-<ver>.<ext>`, darwin uses `arm64`/`x86_64`.

We deliberately did **not** copy that asset naming. depin's pipeline already emits
`<binary>_<goos>_<goarch>.zip` and those exact names are a contract with the fleet -
`cmd/worker-updater/path.go:17-21` substitutes `runtime.GOOS`/`GOARCH` into a templated
URL. Renaming for cosmetics would break auto-update.

## The change

- `scripts/release/publish-github.sh` (new) - curl + jq against api.github.com.
  Idempotent: reuses an existing release for the tag, deletes each asset before
  re-uploading, so a retried job converges. Refuses to run without `sha256sums.sig`.
  `-rc` tags publish as prerelease.
- `.gitlab/ci/release.yml` - `publish:github` job, tag-only, `needs: publish:sign`,
  and `when: never` unless `$DEPIN_GITHUB_RELEASE_REPO` is set (so it merges inert).
  `publish:versioncontrol-config` gained `needs: publish:github` (optional).
- `.gitlab/ci/lint.yml` - new script added to the `lint:release-tooling` tracked-file
  allowlist. That job is what catches a script the `.gitignore` allowlist would drop;
  see [[depin-gitignore-allowlist-gotcha]].
- `scripts/release/compress-binaries.sh` - now also emits an **operator bundle**,
  `aioz-depin-worker_<goos>_<goarch>.zip`, holding worker + worker-updater + keytool +
  a short README.txt, so installing a node is one download instead of three. Built only
  where all three binaries exist, so 32-bit linux/arm (no keytool) is skipped rather
  than shipped incomplete. `.exe` suffixes are preserved inside the Windows bundle.
- `docs/RELEASE.md` §6/§6.1/§7 + the CI-variable table.

Pointing auto-update at GitHub is a **variable, not a code change**:
`DEPIN_RELEASE_BASE_URL=https://github.com/<org>/<repo>/releases/download/$CI_COMMIT_TAG`
feeds `update-versioncontrol.sh --base-url`, which already emits
`${BASE_URL}/worker_{os}_{arch}.zip`. GitHub asset names carry no path separators, so
the template resolves. The updater's keytool-signed auth headers are simply ignored by
GitHub, same as by the fleet's nginx.

## Why the bundle is allowed to hold 3 files and the others are not

`cmd/worker-updater/binary.go:117` rejects any archive without exactly one entry, so
`worker_<platform>.zip` and `worker-updater_<platform>.zip` MUST stay single-entry -
putting keytool in them would fail every fleet auto-update with "archive should contain
only one file". The bundle is exempt purely because nothing fetches it automatically:
the rollout template names the per-binary zips specifically. If anything ever points a
rollout URL at the bundle, it breaks.

## The load-bearing gotcha: require-checksum still cannot be enabled

`Version.URL` is ONE templated string covering every platform, while `Version.Checksum`
is a single scalar (`pkg/version/version.go`). One config entry therefore maps to N
platform zips with N different digests, so there is nowhere to put a correct checksum.
`--version.require-checksum` stays off, and every auto-update installs unverified bytes
(`cmd/worker-updater/checksum.go` warns and proceeds). Fixing it means making the
checksum a per-`{os}_{arch}` value across `pkg/version`, `versioncontrol/config`,
`versioncontrol/api/rollout.go` and `cmd/worker-updater`. Do not assume publishing to
GitHub closed this gap - it only upgraded the transport. See [[worker-autoupdate]].

## Worker CLI facts found while writing the public README

- Binary is `worker`, but its cobra root command is `aioznode`.
- **`worker run` is a no-op stub** (`cmd/worker/start.go` `cmdRun` returns nil). The
  real server command is `worker api`.
- `WorkerConfigCmd` / `SetStorageConfig` (`config storage`) exist but are **never added
  to `rootCmd`** (`cmd/worker/main.go`), i.e. dead code. Do not document them.
- `keytool` has **no `identity` parent command** - it is `keytool create <service>`,
  added straight to root.
- Identity dir defaults **disagree**: keytool uses `<appdir>/aioz/identity`, worker uses
  `<appdir>/aioznode/identity/worker`. Anyone following the defaults ends up with a node
  that cannot find the identity it just minted; always pass `--identity-dir` explicitly.
- `withdraw` is denominated in **USD**, unlike aioz-depin-cli's AIOZ-denominated one.

## Verification status

Verified against a local mock of the GitHub API (`scratchpad/mockgh.py`): create path
(22 assets), retry path (22 replaces, release reused), partial-failure convergence
(4 uploads + 18 replaces after deleting 4 assets server-side), prerelease true/false,
and all seven input guards. `lint:release-tooling` logic passes locally; all CI YAML
parses with every script entry a string.

**Not** verified live: the real publish, the fleet auto-update swap, and a CI tag dry
run. All three are blocked on the same thing - the local `gh` token is `vangxitrum`,
which has `"push": false` on `tuantq-aioz/new-depin` and no AIOZNetwork membership.
