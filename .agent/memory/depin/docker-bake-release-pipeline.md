---
type: fact
tags: [depin, build, release, docker, bake, ci, cross-compile, provenance]
created: 2026-08-05
agent: main
---

# Docker Bake multi-platform release + GitLab CI

Branch `feat/docker-bake-release`, **uncommitted**. Replaces the "build on the host,
`docker save`, scp, `docker load`" flow with a `docker buildx bake` graph, multi-arch
images to ghcr.io + Docker Hub, and a GitLab CI pipeline. Plan:
`Projects/depin/plans/2026-08-05-docker-bake-multiplatform-release-cicd.md`.
Runbook now lives in the repo at `docs/RELEASE.md`. See [[component-version-tags]],
which this builds directly on.

## Shape

- `release.Dockerfile` - multi-stage cross-compiler. Pinned **Zig 0.15.2** (sha256
  verified) as the C toolchain, `FROM --platform=$BUILDPLATFORM` throughout, no QEMU.
- `release.docker-bake.hcl` - `PLATFORMS` map is the single source of truth; matrix
  targets generated from it. `bin-ctx-*` duplicates (cacheonly) feed a
  `combine-platforms` layer so images never recompile.
- `docker-bake.hcl` - dev overlay (host platform, `type=docker`), always passed
  alongside the release file so the matrix is never duplicated.
- `scripts/bake.sh` - computes versions/credentials, then execs bake. Bake HCL cannot
  shell out, so this wrapper is mandatory.
- `scripts/release/{tag-release,check-release-binaries,compress-binaries,build-native,
  update-versioncontrol,sign-artifacts,verify-release}.sh`
- `.gitlab-ci.yml` + `.gitlab/ci/*.yml`; runtime images moved to `cmd/<c>/Dockerfile`.

## Verified matrix (built and metadata-checked, 20 binaries)

linux/amd64 + linux/arm64 (all 7 components), windows/amd64 + freebsd/amd64
(worker, worker-updater, keytool). Linux is **fully static musl** - `readelf -d` shows
no dynamic section at all, which retires the glibc-2.39 pin on the runtime images.
FreeBSD links base-system `libc.so.7`/`libthr.so.3`, same as Storj.

## Traps found the hard way (all cost a full build cycle)

1. **`.dockerignore` glob ate a source package.** `**/hashstore` matches
   `worker/pkg/hashstore`, so the build died with `package aioz-depin/worker/pkg/hashstore
   is not in std`. Never glob a bare package name at any depth. Real runtime state all
   lives under `dev/`. Same latent risk with `**/data`.
2. **`.dockerignore` had never been committed** - the allowlist `.gitignore` swallowed
   it since forever, so a fresh clone built images with no ignore file. Fixed by
   `!*.dockerignore`. Same family as [[depin-gitignore-allowlist-gotcha]].
3. **`git config url.X.insteadOf` REPLACES, it does not append.** Writing the key twice
   kept only the last rule, everything else fell back to HTTPS, and the internal
   GitLab's private CA is not trusted in the build image
   (`certificate signer not trusted`). Must use `git config --add`.
4. **BuildKit's RUN sandbox does not use the host resolver** - it resolves via 8.8.8.8,
   so `gitlab.internal` fails even though the host resolves it. Needs BOTH
   `extra-hosts` in the bake target (bake.sh resolves it via `getent`) and a builder
   created with `--driver-opt network=host`. Go's module discovery hits
   `gitlab.internal/aioz-depin/go-sdk?go-get=1` **by hostname** (the path has no `.git`
   suffix so it cannot be short-circuited), so this is mandatory.
5. **buildx >= 0.23 refuses to read files outside the build context** without
   `--allow=fs.read=<path>`. Hits both the ssh key and the netrc. bake.sh adds the
   entitlement automatically for exactly those two paths.
6. **`go version -m` does NOT record `-ldflags`.** It records `-buildmode`,
   `-compiler`, `-tags`, `-trimpath`, `CGO_ENABLED`, `GOOS`, `GOARCH` and nothing else,
   so storj's `vcs.modified` style check has no analogue here. `check-release-binaries.sh`
   instead greps the binary image for the exact expected version string and executes
   host-platform binaries to parse `<bin> version`.
7. **`-buildvcs=false` is load-bearing.** `pkg/version` prefers Go's embedded VCS data
   over ldflags (`version.go:345-366`), and VCS data is repo-wide, so letting it win
   would flatten per-component versions into one and silently undo
   [[component-version-tags]].
8. **Bake merges `attest` lists across `-f` files instead of replacing.** `attest = []`
   in the dev overlay leaves provenance/sbom in place and a `type=docker` export then
   fails. Must override with `type=provenance,disabled=true`.

## Platform blockers (both real, both outside the build system)

- **macOS cannot be cross-compiled.** Zig handles Go and libsecp256k1 for darwin fine,
  but the dep set needs Apple frameworks - CoreFoundation/CoreServices
  (`99designs/go-keychain` via the cosmos keyring, `rjeczalik/notify`), IOKit
  (gopsutil), `mach/mach_vm.h` (Prometheus process collector). Those headers ship only
  in the macOS SDK, which Apple's licence stops Zig bundling. **User chose a native
  macOS runner**; `scripts/release/build-native.sh` writes into the same
  `release/<ver>/darwin_<arch>/` layout. `DEPIN_MACOS_RUNNER_TAG` unset => job skipped,
  pipeline still green.
- **linux/arm (32-bit) blocked upstream.** `cmd/worker` reaches go-aioz's
  `x/mint/simulation`, `genesis.go:24`: `r.Intn(22143943760)` overflows a 32-bit `int`.
  One-line fix in `10.0.0.50/aioz-network/go-aioz` (`r.Int63n`). Marked `blocked` in
  PLATFORMS. arm64 is unaffected and is what current Pi hardware runs.
  Also needed on 32-bit: `tags = "purego force32bit"`, because curve25519-voi's
  `purego` tag selects the 64-bit generic field impl while excluding the 64-bit types
  (`undefined: feMulGeneric`).

## Own-code portability fixes made along the way

- `pkg/utility/hardwareinfo/hardware_freebsd.go` (new) - the package picks an impl by
  GOOS filename and had no freebsd file. Avoids `jaypipes/ghw` (Linux/Windows only),
  uses gopsutil + `net.Interfaces`. Compile-verified only, never run on FreeBSD.
- `worker/pkg/du/diskusage.go` - `Statfs_t.Bavail` is unsigned on Linux but **signed**
  on FreeBSD/darwin. Added `blocksToBytes`, which normalises through `int64` and clamps
  negatives to 0 rather than wrapping to ~16 EB free.

## Provenance decisions

BuildKit `attest = [provenance mode=max, sbom]` on every image + cosign signatures over
image digests and `sha256sums`. **Keyless is impossible here**: Fulcio must fetch the
OIDC issuer's JWKS over the public internet and `gitlab.internal` is not publicly
resolvable, so `cosign sign --identity-token $CI_JOB_JWT` cannot work. Key pair in
protected CI vars; `cosign.pub` committed. Claim **"L3-aligned"**, not certified L3 -
a self-hosted runner is tenant-controlled.

## Not yet verified

Image push to either registry, cosign signing/verification (cosign and syft are not
installed on this box), the CI pipeline itself, the fleet regression, and running the
arm64/windows/freebsd/darwin binaries on real hardware (no qemu binfmt handler here).
