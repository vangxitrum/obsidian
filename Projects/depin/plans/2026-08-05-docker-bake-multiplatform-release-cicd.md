---
type: plan
project: depin
created: 2026-08-05
updated: 2026-08-07
status: implemented-partial
tags: [depin, build, release, docker-bake, ci-cd, cross-compile, provenance]
---

# Multi-platform release: Docker Bake + multi-stage Dockerfile + GitLab CI/CD

## Context

Today the depin build is a laptop ritual. `Makefile.build` compiles one binary at a time for
the host arch only (`GOOS ?= go env GOHOSTOS`), the three Dockerfiles are runtime-only
`FROM ubuntu:24.04` shells that `COPY bin/<binary>` from the host, distribution is
`docker save` -> scp -> `docker load`, and there is no CI at all (`.gitlab-ci.yml` does not
exist; `DEVELOPING.md` says `## Continuous Integration - TODO:`). Every image is pinned to
glibc 2.39 because the binaries are dynamically linked against the build host's libc.

We want what Storj has: one platform matrix expressed as data, cross-compiled binaries and
multi-arch images produced by `docker buildx bake`, published to public registries
(ghcr.io + Docker Hub) so worker operators can `docker pull` instead of receiving a 224MB zip.
On top of Storj we want supply-chain provenance, which Storj does not do at all.

Three findings shape the design:

1. **`worker`, `worker-updater` and `keytool` require cgo.** `ethermint` -> `go-ethereum/crypto/secp256k1`
   is cgo-only; `CGO_ENABLED=0` fails with `undefined: secp256k1.RecoverPubkey` even with `-tags purego`.
   `coord`, `edgeserver` and `versioncontrol` build cleanly with `CGO_ENABLED=0`. So the binaries we
   most want everywhere are exactly the ones that need a per-target C toolchain.
   Storj solves this with **Zig** (`release.Dockerfile:14-29`, `release.docker-bake.hcl:49-116`):
   `zig cc -target <triple>` cross-compiles cgo for linux-musl, windows-gnu, freebsd and macos
   from a single linux/amd64 host, no QEMU, no osxcross. We adopt that verbatim.

2. **Private Go deps are LAN-only.** `go.mod:385-402` replaces cosmos-sdk, ethermint, ibc-go,
   Gravity-Bridge and go-sdk onto `10.0.0.50` / `gitlab.internal` (same host). No cloud runner can
   fetch them. CI runs on the internal self-hosted runner, and BuildKit needs credentials + DNS
   for that host injected as a build secret.

3. **`.gitignore` is an allowlist** (`* ` then `!`-entries). Every new non-`.go` file
   (`*.hcl`, `.gitlab-ci.yml`, `release.Dockerfile`, `scripts/release/*.sh`) is silently dropped
   unless explicitly allowlisted. This has bitten this repo twice already.

Decisions taken by the user: CI on the existing internal GitLab runner; publish to **ghcr.io
and Docker Hub**; **full Storj platform parity including macOS**; **full SLSA L3** provenance.

---

## Target graph

`docker buildx bake` builds a DAG of targets. `_`-prefixed targets are abstract (inherited, never
built directly). Everything below is generated from one `PLATFORMS` map, so adding a platform is
one map entry.

```
                        ┌──────────────┐
                        │ _common      │  context=., dockerfile=release.Dockerfile,
                        │ (abstract)   │  ssh/secrets, BUILD_* args, cache-from/to
                        └──────┬───────┘
                               │ inherits
        ┌──────────────────────┼──────────────────────────────────────┐
        │                      │                                      │
  matrix over PLATFORMS (7)    │                            (future) web UI targets
        │                      │                                      │
 ┌──────▼───────────────┐  ┌───▼──────────────────┐        ┌──────────▼──────────┐
 │ binaries-linux-amd64 │  │ bin-ctx-linux-amd64  │        │ web-console         │
 │ binaries-linux-arm64 │  │ bin-ctx-linux-arm64  │        │ (node build ->      │
 │ binaries-linux-arm   │  │ bin-ctx-linux-arm    │        │  FROM scratch AS    │
 │ binaries-windows-... │  └───┬──────────────────┘        │  export)            │
 │ binaries-freebsd-... │      │ named contexts            └──────────┬──────────┘
 │ binaries-macos-amd64 │      │                                      │ context into
 │ binaries-macos-arm64 │  ┌───▼───────────────┐                      │ _common (go:embed)
 └──────┬───────────────┘  │ binaries-linux    │                      │
        │                  │ target=combine-   │◄─────────────────────┘
        │ output=type=local│ platforms         │
        │ ->release/$VER/  │ /linux_amd64/...  │
        │   <os>_<arch>/   │ /linux_arm64/...  │
        ▼                  │ /linux_arm/...    │
   [ zip + sha256sums ]    └───┬───────────────┘
   [ syft SBOM        ]        │ contexts = { binaries = "target:binaries-linux" }
   [ cosign sign-blob ]        │
                    ┌──────────┼──────────┬──────────────┬─────────────────┐
                    │          │          │              │                 │
            ┌───────▼──┐ ┌─────▼─────┐ ┌──▼──────────┐ ┌─▼────────────┐ ┌──▼────────┐
            │coord-    │ │worker-    │ │edgeserver-  │ │versioncontrol│ │keytool-   │
            │image     │ │image      │ │image        │ │-image        │ │image      │
            │amd64,    │ │amd64,arm64│ │amd64,arm64  │ │amd64,arm64   │ │amd64,arm64│
            │arm64     │ │,arm       │ │             │ │              │ │,arm       │
            └───────┬──┘ └─────┬─────┘ └──┬──────────┘ └─┬────────────┘ └──┬────────┘
                    └──────────┴──────────┴──────────────┴─────────────────┘
                                          │ attest = provenance mode=max + sbom
                                          │ tags   = ghcr.io/<org>/x  AND  docker.io/<org>/x
                                          ▼
                                    [ --push ] -> [ cosign sign + attest ]

groups:
  binaries = all 7 binaries-* targets            (make release/binaries/build)
  images   = the 5 *-image targets               (make release/images/build|push)
  release  = binaries + images                   (tag pipeline)
  dev      = single-platform, type=docker load   (docker-bake.hcl, local iteration)
```

Two properties matter and both come straight from Storj:

- **Build once, package many.** The `bin-ctx-*` targets export to `scratch` with no output;
  `binaries-linux` fans them into `/linux_amd64`, `/linux_arm64`, `/linux_arm`; every runtime
  Dockerfile then does `COPY --from=binaries /${TARGETOS}_${TARGETARCH}/<bin>`. Five images
  across three arches recompile nothing.
- **The matrix is data.** `PLATFORMS` is a map of `{goos, goarch, cgo, cc, cpp, ldflags, components}`.
  Per-component platform coverage is expressed by which platforms list that component.

### Platform matrix (Storj parity, adjusted for our cgo reality)

| component | linux/amd64 | linux/arm64 | linux/arm | windows/amd64 | freebsd/amd64 | macos/amd64 | macos/arm64 |
|---|---|---|---|---|---|---|---|
| `worker` (= storagenode) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `worker-updater` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `keytool` (= identity) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `coord` (= satellite) | ✓ | ✓ | - | - | - | - | - |
| `versioncontrol` | ✓ | ✓ | - | - | - | - | - |
| `edgeserver` (= gateway) | ✓ | ✓ | - | - | - | - | - |
| `relay-ping` | ✓ | ✓ | - | - | - | - | - |

Storj's shape for the server side (`satellite` on 2 platforms, `identity` on all 7), and
**wider than Storj on the node side**: Storj does not ship `storagenode` for macOS at all, we do.
`worker` and `worker-updater` are on all 7 platforms, so a macOS operator can run a node natively.

`uplink` is deliberately excluded: `cmd/uplink` does not compile today (`package aioz-depin/uplink is
not in std` - the `uplink/` directory does not exist), a pre-existing breakage unrelated to this work.
Add it to the matrix in the same one-line way once it builds.

cgo/toolchain per platform:

| platform | CGO | CC | extra ldflags |
|---|---|---|---|
| linux/amd64 | 1 | `zig cc -target x86_64-linux-musl` | `-linkmode external -extldflags "-static"` |
| linux/arm64 | 1 | `zig cc -target aarch64-linux-musl` | same |
| linux/arm | 1 | `zig cc -target arm-linux-musleabi` | same |
| windows/amd64 | 1 | `zig cc -target x86_64-windows-gnu` | - |
| freebsd/amd64 | 1 | `zig cc -target x86_64-freebsd-none` | - |
| macos/amd64 | 1 | `zig cc -target x86_64-macos-none` | - |
| macos/arm64 | 1 | `zig cc -target aarch64-macos-none` | - |

Static musl on linux is a real win beyond portability: it retires the glibc-2.39 coupling
documented in `Dockerfile.coord:19-20` and `dev/fleet/Dockerfile.fleet:4-5`, so runtime images
can drop from `ubuntu:24.04` to `debian:trixie-slim` (or distroless).

**Risk, called out up front - the macOS rows are the one unproven part of this plan.** Storj sets
`cgo = "0"` for both macOS rows because the only components they build for macOS (`uplink`,
`identity`) are pure Go. Ours are not: `worker`, `worker-updater` and `keytool` all pull in
`go-ethereum/crypto/secp256k1`, so macOS means cross-compiling **cgo** for darwin, which Storj's
setup never exercises and which is the hardest cell in any cross-compile matrix.

Zig does bundle macOS libc headers and libSystem `.tbd` stubs, so `zig cc -target aarch64-macos-none`
with `CGO_ENABLED=1` is expected to work, and `libsecp256k1` is vendored C inside the go-ethereum
module (no external library to obtain). But this must be proven, not assumed. Step 3 runs it as an
explicit spike with a fallback ladder, in order:

1. **Zig** (`zig cc -target <arch>-macos-none`) - preferred; keeps one builder image, no extra runner.
2. **osxcross + a macOS SDK** in the `build-tools` stage - still one Linux builder, but adds an
   SDK we must legally obtain and store, and a heavier image.
3. **A native macOS runner** (a Mac mini registered as a second GitLab runner, or GitLab's SaaS
   macOS runners) building the two darwin rows and handing artifacts back as job artifacts.
   The bake graph does not change: the darwin `binaries-macos-*` targets are simply produced by
   a different job and merged into the same `release/<ver>/macos_<arch>/` layout.

The spike decides which rung we land on; it does not decide *whether* macOS ships. Nothing in
steps 4-11 depends on the outcome, so the rest of the work proceeds in parallel.

One consequence to plan for either way: **macOS binaries downloaded by a human get quarantined by
Gatekeeper** unless they are codesigned with a Developer ID and notarized. The auto-update path is
unaffected (the updater writes the binary itself, so no quarantine xattr is set), but first-time
manual install is. See "Out of scope" - notarization needs an Apple Developer account and is
tracked as a follow-up, with `docs/RELEASE.md` documenting the
`xattr -d com.apple.quarantine` workaround until then.

---

## Files

New:

- `release.Dockerfile` + `release.Dockerfile.dockerignore` - the cross-compiler, multi-stage.
- `release.docker-bake.hcl` - matrix, binaries, images, attestations, dual-registry tags.
- `docker-bake.hcl` - dev overlay: single platform, `type=docker` output, no attestations.
- `docker-bake-ci.hcl` - CI overlay adding `cache-to=type=registry` (main branch only).
- `Makefile.release.mk` - `release/*` targets, included from `Makefile`.
- `scripts/bake.sh` - exports per-component `BUILD_VERSION_*` + `TAG`/`LATEST_TAG`, then `docker buildx bake "$@"`.
- `scripts/release/{check-release-binaries,compress-binaries,sign-artifacts,verify-release,publish-release,update-versioncontrol}.sh`
- `cmd/{coord,worker,edgeserver,versioncontrol,keytool}/Dockerfile` - runtime images.
- `cmd/{coord,worker,edgeserver}/entrypoint` - only where the current image has non-trivial startup.
- `.gitlab-ci.yml` + `.gitlab/ci/{lint,test,build,release}.yml`.
- `docs/RELEASE.md` - the release runbook (step 11).

Modified: `.gitignore` (allowlist), `Makefile`, `Makefile.build`, `MAINTAINERS.md`, `DEVELOPING.md`.

Deleted at the end: `Dockerfile.coord`, `Dockerfile.edgeserver`, and the `build-*-image` /
`save-*-image` targets they back.

Reused as-is (do not reinvent): `scripts/component-version.sh` and `scripts/source-version.sh`
(both already honour `BUILD_VERSION` / `SOURCE_VERSION` overrides for exactly this purpose),
the `pkg/version` ldflag set at `pkg/version/version.go:31-35`, and
`cmd/worker-updater/path.go:18-19` which already substitutes `{os}`/`{arch}` into rollout URLs -
so the existing versioncontrol channel becomes multi-platform for free once CI produces
`worker_<os>_<arch>.zip`.

---

## Steps

### 1. Repo prep

Add to `.gitignore` before anything else, or the work is invisible to a fresh clone:

```
!*.hcl
!release.Dockerfile
!*.dockerignore
!cmd/*/Dockerfile
!cmd/*/entrypoint
!.gitlab-ci.yml
!.gitlab/**
!scripts/release/*.sh
```

Verify with `git status --short` after creating a stub of each file, and with
`git ls-files --others --ignored --exclude-standard | grep -E '\.hcl|gitlab'` returning nothing.

### 2. `release.Dockerfile` - the multi-stage cross-compiler

Mirrors `storj/release.Dockerfile`, plus the private-module handling Storj does not need.

```
# syntax=docker/dockerfile:1.7-labs
ARG GO_VERSION=1.25
ARG ZIG_VERSION=0.15.2

FROM --platform=$BUILDPLATFORM golang:${GO_VERSION} AS build-tools
  apt-get: build-essential wget xz-utils git ca-certificates zip
  download Zig for $BUILDPLATFORM, verify pinned sha256, PATH=/usr/local/zig

FROM build-tools AS download-dependencies
  ENV GOPRIVATE/GOINSECURE/GONOSUMDB = gitlab.internal/*,10.0.0.50/*
  git config url."https://gitlab-ci-token:<secret>@gitlab.internal/".insteadOf ...
  COPY go.mod go.sum
  RUN --mount=type=secret,id=netrc,target=/root/.netrc \
      --mount=type=cache,target=/go/pkg/mod  go mod download

FROM download-dependencies AS build-binaries
  ARG GOOS GOARCH CGO_ENABLED CC CXX GO_LDFLAGS COMPONENTS
  ARG COMPONENT_VERSIONS SOURCE_VERSION BUILD_COMMIT BUILD_TIMESTAMP BUILD_RELEASE
  COPY . /work/
  RUN --mount=type=cache,target=/root/.cache/go-build --mount=type=cache,target=/go/pkg/mod \
      for c in $COMPONENTS; do \
        name=$(basename $c); \
        ver=$(echo "$COMPONENT_VERSIONS" | tr ';' '\n' | grep "^$name=" | cut -d= -f2); \
        go build -trimpath -o /out/ \
          -tags purego \
          -ldflags "-s -w ${GO_LDFLAGS} \
            -X aioz-depin/pkg/version.buildVersion=$ver \
            -X aioz-depin/pkg/version.buildSource=$SOURCE_VERSION \
            -X aioz-depin/pkg/version.buildCommitHash=$BUILD_COMMIT \
            -X aioz-depin/pkg/version.buildTimestamp=$BUILD_TIMESTAMP \
            -X aioz-depin/pkg/version.buildRelease=$BUILD_RELEASE" \
          "$c"; \
      done

FROM scratch AS export-binaries
COPY --from=build-binaries /out/* /

FROM scratch AS combine-platforms
COPY --from=linux_amd64 /* /linux_amd64/
COPY --from=linux_arm64 /* /linux_arm64/
COPY --from=linux_arm   /* /linux_arm/
```

Two deliberate divergences from Storj:

- **Per-component versions survive.** Storj injects only `buildRelease` and lets Go VCS build info
  supply the rest, which forces one version for the whole repo. We already have something better
  (`MAINTAINERS.md:3-25`), so the `RUN` loops over components and looks its version up in a
  `COMPONENT_VERSIONS` build arg of the form `coord=v0.0.2;worker=v0.0.0-dev.…;…`. One `RUN` layer,
  cache behaviour unchanged.
- **`-trimpath` is added** (absent from `Makefile.build` today). It removes host paths from the
  binary, which both shrinks the artifact and is a prerequisite for reproducible builds under SLSA.

Also note this switches the two file-list builds (`./cmd/coord/*.go`, `./cmd/worker/*.go`,
`./cmd/edgeserver/*.go`) to package paths. That re-enables Go's `-buildvcs` stamping, so
`pkg/version` will start preferring VCS data over our ldflags for timestamp/commit
(`version.go:345-366`). That is the desired behaviour and matches Storj, but it means the
worktree must be clean in CI - enforced in step 8.

### 3. `release.docker-bake.hcl`

- `variable "PLATFORMS"` - the map from the matrix table above.
- `variable "BUILD_VERSION"`, `"SOURCE_VERSION"`, `"BUILD_COMMIT"`, `"BUILD_TIMESTAMP"`,
  `"BUILD_RELEASE" = "true"`, `"TAG"`, `"LATEST_TAG"`, `"CUSTOMTAG"`, `"REGISTRIES"`.
- `target "_common"` - context `.`, `dockerfile = "release.Dockerfile"`, the shared args,
  `secret = ["id=netrc,env=NETRC"]`.
- Matrix-generated `binaries-<os>-<arch>` (target `export-binaries`,
  `output = ["type=local,dest=release/${BUILD_VERSION}/${os}_${arch}"]`) and `bin-ctx-<os>-<arch>`
  (same target, `output = []`).
- `target "binaries-linux"` with `contexts` fanning the three linux `bin-ctx-*` in, `target = "combine-platforms"`.
- Five `*-image` targets, each `dockerfile = "cmd/<c>/Dockerfile"`,
  `contexts = { binaries = "target:binaries-linux" }`, its own `platforms` list,
  OCI `labels` (`org.opencontainers.image.{source,revision,version,licenses}`), and
  `attest = ["type=provenance,mode=max", "type=sbom"]`.
- `function "image_tags"` emitting **both** registries per tag:
  `ghcr.io/${GHCR_ORG}/${name}:${TAG}`, `docker.io/${DOCKERHUB_ORG}/${name}:${TAG}`,
  plus the `${LATEST_TAG}` variants when set. One build, two registries, one push.
- Groups `binaries`, `images`, `release`.

**Spike gate here, and it is a blocking one for macOS only.** Before wiring the macOS rows into
the `binaries` group, run `docker buildx bake -f release.docker-bake.hcl binaries-macos-arm64`
standalone with `CGO_ENABLED=1`. Success criterion is not "the build exits 0" - it is that the
resulting `worker` is a Mach-O arm64 binary that **runs on a real Mac** and prints its version
(see Verification item 4). If Zig fails to link, walk the fallback ladder from the matrix section
(osxcross, then a native macOS runner) rather than shrinking the matrix. Record which rung we
landed on in `docs/RELEASE.md`, since it changes what a maintainer has to keep working.
Do not let this block steps 4-11.

### 4. `scripts/bake.sh`

Bake HCL cannot shell out; Storj solves this the same way (`storj/scripts/bake.sh`). Ours exports:

```
SOURCE_VERSION=$(./scripts/source-version.sh)
BUILD_COMMIT=$(git rev-parse --short HEAD)
BUILD_TIMESTAMP=$(git log -1 --format=%ct)
BUILD_VERSION=${GIT_TAG:-v0.0.0-dev.$BUILD_TIMESTAMP.g$BUILD_COMMIT}
COMPONENT_VERSIONS="coord=$(./scripts/component-version.sh coord);worker=$(...);…"
TAG / LATEST_TAG  per Storj's branch logic (exact tag -> TAG=tag, LATEST_TAG=""; main -> TAG=sha,
                  LATEST_TAG=latest; release-* -> TAG=sha-branch, LATEST_TAG=branch-latest)
exec docker buildx bake "$@"
```

### 5. Runtime Dockerfiles

One per published component, all the same shape, all consuming the `binaries` context:

```
# syntax=docker/dockerfile:1.7-labs
FROM --platform=$BUILDPLATFORM debian:trixie-slim AS ca-cert
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates && update-ca-certificates

FROM debian:trixie-slim
ARG TARGETOS TARGETARCH
COPY --from=ca-cert /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=binaries /${TARGETOS}_${TARGETARCH}/coord   /usr/local/bin/coord
COPY --from=binaries /${TARGETOS}_${TARGETARCH}/keytool /usr/local/bin/keytool
...
```

Preserve the semantics the current files already encode, do not redesign them:
`Dockerfile.coord:25-45` (uid 10001 `coord` user, `VOLUME /data`, `EXPOSE 7777 7779 7781 7788`,
`ENTRYPOINT ["coord"]`, `CMD ["run", "--config-dir", "/data", "--identity-dir", "/data/identity"]`),
`Dockerfile.edgeserver` equivalently on 8888. Keep `USER 10001` - Storj runs everything as root,
we should not copy that.

`cmd/worker/Dockerfile` is the new one and matters most, since it is what the public pulls.
It carries `worker` + `worker-updater` + `keytool` and reuses `dev/fleet/entrypoint.sh` (183 lines,
already handles the updater/supervisor split and the `ENABLE_AUTO_UPDATE=false` redeploy-as-update
mode). Move it to `cmd/worker/entrypoint`, keep the `FLEET_*` ARG/ENV baking from
`dev/fleet/Dockerfile.fleet:27-52` since `dev/fleet/worker-pkg/init.sh` reads those via
`docker inspect`. Note it is a bash script, so the base image needs bash (`debian:trixie-slim` has it).
`dev/fleet/Dockerfile.fleet` stays as the dev/fleet-test image and is retargeted to
`FROM ghcr.io/<org>/worker:<tag>` once the real image exists.

### 6. Packaging and checksums

`scripts/release/compress-binaries.sh`, modelled on Storj's: one zip per binary per platform,
named `<component>_<goos>_<goarch>.zip` with the binary at the zip root (single entry - required,
`cmd/worker-updater/binary.go:105` rejects any zip that does not have exactly one file), then a
single sorted `sha256sums`. This naming is what `cmd/worker-updater/path.go:18-19` already expects
from `{os}`/`{arch}` templating, so `scripts/release/update-versioncontrol.sh` can rewrite
`dev/fleet/versioncontrol.config.yaml` to a single templated URL
(`https://<host>/<version>/worker_{os}_{arch}.zip`) and stop being amd64-only.

**Naming trap to get right the first time:** Storj keys its matrix on `macos/amd64` but sets
`goos = "darwin"`, so their release directory and their artifact name can disagree. Our updater
substitutes `runtime.GOOS`, which is `darwin`, never `macos` (`cmd/worker-updater/path.go:18`).
Name every output directory and zip from `${goos}_${goarch}` - `darwin_arm64`, not `macos_arm64` -
and keep `macos/*` only as the human-facing matrix key. A mismatch here produces a 404 that only
appears on macOS nodes at rollout time, which is the worst possible moment to find it. The dry-run
in Verification item 10 must exercise a darwin URL specifically.

### 7. Provenance and signing (SLSA L3 target)

Four layers, in increasing cost:

**a. Build integrity.** Pin `GO_VERSION` and `ZIG_VERSION` with sha256 verification on the Zig
tarball (Storj does this, `release.Dockerfile:19-25`); pin base images by digest; `-trimpath`;
`SOURCE_DATE_EPOCH` from the tag commit; cache mounts used only for caches, never for correctness.

**b. Attestations on images.** `attest = ["type=provenance,mode=max", "type=sbom"]` per image
target. BuildKit emits an in-toto SLSA provenance predicate and an SPDX SBOM as attestation
manifests in the image index. Both ghcr.io and Docker Hub store and display these. Verify with
`docker buildx imagetools inspect --format '{{json .Provenance}}'`.

**c. Attestations on binaries.** `syft` SBOM per platform directory, plus GitLab's own
`RUNNER_GENERATE_ARTIFACTS_METADATA=true` which emits SLSA provenance for job artifacts.

**d. Signing.** `cosign sign` on every pushed image digest, `cosign attest --type slsaprovenance`
for the predicate, `cosign sign-blob` over `sha256sums`.

> **Honest constraint on "keyless".** Sigstore keyless signing needs Fulcio to fetch the OIDC
> issuer's JWKS over the public internet. `gitlab.internal` is not publicly resolvable, so it
> cannot be a Fulcio-trusted issuer, and `cosign sign --identity-token $CI_JOB_JWT` will fail.
> Use **cosign with a private key held in a protected+masked GitLab CI variable**
> (`COSIGN_KEY` + `COSIGN_PASSWORD`), generated offline, with `cosign.pub` committed to the repo
> and documented in `MAINTAINERS.md`. Keyless becomes available only if we later publish from a
> GitHub Actions mirror, where `actions/attest-build-provenance` gives signed SLSA L3 provenance
> for free - worth revisiting when public distribution is real, but out of scope here.

> **Honest constraint on "L3".** SLSA L3 requires provenance generated by a build service the
> tenant cannot forge. On a self-hosted runner we can satisfy every mechanical requirement -
> scripted build, ephemeral isolated environment, provenance emitted by BuildKit rather than by
> our own script, secrets scoped to protected refs - but a maintainer with runner access could in
> principle forge it. What we can honestly claim is **L3-aligned**, and the plan should say so in
> `MAINTAINERS.md` rather than overclaim. To get as close as possible: release jobs run **only on
> protected tags**, on a runner configured with an ephemeral executor (fresh container per job,
> `FF_NETWORK_PER_BUILD`), with the signing key and registry credentials available on protected
> refs only, and tag creation restricted to maintainers.

`scripts/release/verify-release.sh` closes the loop: `cosign verify`,
`cosign verify-attestation --type slsaprovenance`, `imagetools inspect` for the SBOM, and a
`sha256sum -c`. It runs as the last CI stage against what was actually pushed, so a broken
attestation fails the release rather than being discovered by a user.

### 8. `.gitlab-ci.yml`

Runner prerequisites (one-time, outside the repo, document in `DEVELOPING.md`):
buildx `docker-container` driver created with `--driver-opt network=host` so BuildKit can resolve
`gitlab.internal`; runner tagged e.g. `depin-internal`; egress to ghcr.io / docker.io.

```
stages: [lint, test, build, image, publish, verify]

variables:
  GOPRIVATE/GOINSECURE: gitlab.internal/*,10.0.0.50/*
  RUNNER_GENERATE_ARTIFACTS_METADATA: "true"

lint          : make lint                                        (MR + main)
test          : go test ./... with a postgres:18.1-alpine service (MR + main)
build:binaries: scripts/bake.sh -f release.docker-bake.hcl binaries
                then check-release-binaries.sh, compress-binaries.sh
                artifacts: release/$BUILD_VERSION/**            (main + tags)
image:build   : scripts/bake.sh -f release.docker-bake.hcl images            (MR: build only)
image:push    : scripts/bake.sh -f release.docker-bake.hcl images --push     (main + tags)
                + docker-bake-ci.hcl overlay for registry cache on main
publish:sign  : cosign sign / attest / sign-blob                 (tags only, protected)
publish:release: GitLab Release + zips into the generic package registry
                 + update-versioncontrol.sh emitting the new rollout config  (tags only)
verify        : verify-release.sh                                (tags only)
```

MR pipelines stop at `image:build` (no push, no secrets). Only protected tags reach `publish:*`.

### 9. Wire into Make, retire the old path

`Makefile.release.mk` with Storj's target names so the two repos stay comparable:
`release/info`, `release/binaries/build`, `release/binaries/check-release`, `release/binaries/compress`,
`release/images/build`, `release/images/push`, `release/images/clean`.
`Makefile.build` keeps `build-*` for fast host-native dev builds (unchanged), and
`build-coord-image` / `build-edgeserver-image` / `build-fleet-image` become thin aliases onto
`docker bake -f docker-bake.hcl <target>`. `save-*-image` stays (the scp path is still how coord
gets to the VPS today). Delete `Dockerfile.coord` and `Dockerfile.edgeserver` only after the bake
images are verified running.

### 10. Web UI hook (reserved, not built)

Storj's pattern is worth adopting now as an empty seam so the graph does not need restructuring
later: a UI gets `web/<app>/Dockerfile` ending in

```
FROM --platform=$BUILDPLATFORM node:${NODE_VERSION} AS build
RUN --mount=type=cache,target=/root/.npm npm ci && npm run build
FROM scratch AS export
COPY --from=build /work/dist /
```

and is wired as a **named context**. Two consumption modes, both already exercised by Storj:
embedded (`contexts` into `_common`, `COPY --from=web-x / /work/web/x/dist/` before `go build`,
so `//go:embed` picks it up - Storj does this for storagenode/multinode) or loose files
(`contexts` into the runtime image target, served from disk - Storj does this for the satellite
console). Because it is `--platform=$BUILDPLATFORM` and exports to `scratch`, the UI is built once
and shared by all 7 target platforms. Add `variable "NODE_VERSION"` and a commented
`# "web-console" = "target:web-console"` line in `_common.contexts` now; nothing else.

### 11. `docs/RELEASE.md` - the release runbook

The pipeline is only half the deliverable. `MAINTAINERS.md` currently has three literal
`TODO`s (`:55-57` changelog, `:59-61` cutting a release branch, `:83-85` where the binaries are)
and references a `branch.py` that does not exist in this repo. Write the real document, and
replace those TODOs with links into it.

`docs/RELEASE.md` covers, in order:

1. **What a version means here.** Point at the existing `MAINTAINERS.md:3-25` model rather than
   restating it: repo-level `Source:` tag vs per-component `Version:`, why dev versions
   (`v0.0.0-dev.<ts>.g<hash>`) sort below every release tag, and the consequence - only an exact
   repo tag makes all components share one version, so **releases are always cut from a tag**.
2. **Release cadence and branch model.** `develop` -> `main`; `release-<major>.<minor>` branches
   for maintenance; hotfixes cherry-picked onto the release branch, never reverted from it
   (`MAINTAINERS.md:77-81` already states this policy - keep it, just make it concrete).
3. **Pre-flight checklist.** Worktree clean (CI enforces `vcs.modified=false`, step 8, and a
   dirty tree silently produces a `-dirty` source tag); `make lint` and the test suite green on
   the target commit; migrations reviewed; `CHANGELOG`/release note drafted via the existing
   `release-notes` skill, which writes to `.gitlab/releases/` - **that directory does not exist
   yet**, so create it and allowlist it in `.gitignore` as part of this step.
4. **Cutting the release.** The exact commands, in order:
   ```
   git checkout main && git pull
   make version-info                      # sanity: what each component will be stamped with
   scripts/release/tag-release.sh v0.0.3  # refuses on a dirty tree or a malformed version
   git push origin v0.0.3                 # protected tag -> triggers the release pipeline
   ```
   Add `scripts/release/tag-release.sh` (port of Storj's, 30 lines: semver regex gate +
   clean-worktree gate + `git tag`) to the file list in step 6 - it is the one manual command
   in the whole flow, so it should be the one that refuses to do the wrong thing.
5. **What CI does, and how to watch it.** The stage list from step 8, what each produces, and
   the expected wall-clock. Which stages are safe to retry and which are not (`publish:release`
   is not idempotent against an existing GitLab release; `image:push` is).
6. **Where the artifacts land.** Concretely, with URLs: GitLab Release page + generic package
   registry for the `<component>_<goos>_<goarch>.zip` set and `sha256sums`;
   `ghcr.io/<org>/{coord,worker,edgeserver,versioncontrol,keytool}` and the Docker Hub mirror;
   which tags exist (`v0.0.3`, plus `latest` only from `main`, plus `release-0.0-latest` from a
   release branch). This replaces the `MAINTAINERS.md:83-85` TODO.
7. **Rolling out to workers.** Two independent channels, and the doc must be explicit that they
   are independent:
   - **Registry channel** - operators running the Docker image pull the new tag. With
     `ENABLE_AUTO_UPDATE=false` (the current fleet default, see `dev/fleet/entrypoint.sh:24-31`)
     redeploying the image *is* the update.
   - **versioncontrol channel** - `scripts/release/update-versioncontrol.sh` publishes the new
     `suggested.version` + templated `{os}`/`{arch}` URL, then the rollout cursor is advanced.
     Document the cursor semantics (`seed`, `previous_cursor`, `cursor` as a percentage,
     `safe_rate` ramp) and a staged schedule: 0 -> 5 -> 25 -> 100 with a soak between steps.
     Note the live-fleet gotcha already on record: those containers were started with an
     unreachable `VC_SERVER_ADDRESS`, so their auto-update is inert and they only move via the
     registry channel until that is fixed.
8. **Promoting to `latest`.** `MAINTAINERS.md:87-89` already says the new release becomes
   "Latest" only after 100% rollout. Make it an action: which tag to move, and the
   `docker buildx imagetools create` one-liner that re-points `:latest` at an existing digest
   without rebuilding.
9. **Verifying a release as an operator.** The public-facing half of step 8's verification:
   `cosign verify --key cosign.pub`, `cosign verify-attestation --type slsaprovenance`,
   `sha256sum -c sha256sums`, and where `cosign.pub` lives. This is the only part of the
   provenance work that end users ever touch, so it belongs in the doc, not just in CI.
10. **Rollback.** Roll the versioncontrol cursor back and re-point `suggested` at the previous
    version; re-tag `:latest` to the previous digest; when a bad binary is already deployed,
    the minimum-version lever (`minimum.version`) forces an immediate downgrade-or-stop.
    State plainly that yanking a published tag from a public registry is not a rollback -
    the fix is to publish a superseding version.
11. **Hotfix / cherry-pick flow.** Keeps `MAINTAINERS.md:63-75`'s existing wording, made
    executable against the release branch, ending in a point-release tag.
12. **Troubleshooting table.** The failure modes this repo has actually hit, so nobody
    re-debugs them: dirty worktree -> `-dirty` source tag / `check-release` failure;
    a new file missing from the allowlist -> empty version or a fresh clone that will not build;
    `go mod download` failing in BuildKit -> netrc secret or builder network;
    a zip with more than one entry -> updater rejects it;
    version mismatch between the zip's binary and `suggested.version` -> updater refuses the swap.

Then edit `MAINTAINERS.md`: fill the three TODOs with one-line pointers into `docs/RELEASE.md`,
delete the `branch.py` section (that script does not exist here), and add the SLSA posture
statement from step 7.

---

## Verification

Each step verified before the next; nothing is claimed working from inspection alone.

1. **Allowlist** - `git status --short` shows every new file as untracked-and-addable; a
   `git clone` of a test bundle contains `release.docker-bake.hcl` and `scripts/release/`.
2. **Single-platform binaries** - `./scripts/bake.sh -f release.docker-bake.hcl binaries-linux-amd64`
   produces `release/<ver>/linux_amd64/{coord,worker,worker-updater,keytool,versioncontrol,edgeserver,relay-ping}`.
   `file release/*/linux_amd64/worker` reports `statically linked`.
   `./release/<ver>/linux_amd64/worker version` prints the same `Version:` / `Source:` /
   commit as `make version-info` for that component, and the versions **differ between components**
   when HEAD is not an exact tag (that is the whole point of `component-version.sh`).
3. **Cross-compilation** - `docker bake binaries` builds all 7 platforms.
   `file` on each output reports the expected arch/format (ELF arm64, PE32+ for windows,
   Mach-O for macos, FreeBSD ELF). Run the arm64 worker under `qemu-aarch64-static` (or on a real
   Pi) and confirm `worker version` exits 0 - proves the musl static link, which `file` alone
   does not.
4. **macOS** - the strictest check in the plan, because nothing else covers darwin+cgo.
   `file` reports `Mach-O 64-bit executable arm64`; then copy `worker`, `worker-updater` and
   `keytool` to an actual Mac (Apple Silicon and, if reachable, Intel) and confirm
   `./worker version` and `./keytool --help` exit 0 - a darwin binary that links but crashes on
   a missing libSystem symbol only shows up at runtime. Then run `worker setup` + a check-in
   against a dev coordinator so we know the node actually works, not just that it starts.
   If any of this fails, walk the fallback ladder; do not ship the artifact.
5. **Images** - `docker bake images` then `docker buildx imagetools inspect ghcr.io/<org>/worker:<tag>`
   lists linux/amd64, linux/arm64, linux/arm plus attestation manifests.
   `docker run --rm ghcr.io/<org>/coord:<tag> version` and the same for worker/edgeserver.
6. **End-to-end runtime** - bring the coord image up against the existing dev compose
   (`docker-compose.yml` postgres + `make setup-coord`) and run one upload/download through
   `dev/bench/download-test.sh` or the existing testplanet e2e. The image is only good if a real
   transfer completes, not if the binary prints a version.
7. **Fleet regression** - repoint `dev/fleet/Dockerfile.fleet` at the published worker image,
   run `dev/fleet/fleet.sh build` + a 2-worker `workers-up`, and confirm check-in against the
   coordinator. This is the path the 50 live fleet workers use; it must not regress.
8. **Provenance** - `cosign verify --key cosign.pub ghcr.io/<org>/worker:<tag>`,
   `cosign verify-attestation --type slsaprovenance`, and
   `docker buildx imagetools inspect --format '{{json .SBOM}}'` all succeed. Then deliberately
   break one (re-push an unsigned tag) and confirm `verify-release.sh` fails - an unverified
   verifier is worthless.
9. **Pipeline** - open an MR: lint/test/image-build run, no push, no secrets exposed.
   Push a throwaway protected tag `v0.0.3-rc1`: full release runs, artifacts land in the GitLab
   release + both registries, `verify` is green. Delete the tag and the test tags afterwards.
10. **Updater** - point a worker at the CI-produced `worker_linux_amd64.zip` through a local
    versioncontrol server and confirm the auto-update swap still works
    (`cmd/worker-updater` rejects a version mismatch, so this also re-proves the stamping).
11. **Runbook** - the `v0.0.3-rc1` dry run in item 9 is executed *by following `docs/RELEASE.md`
    verbatim*, not from memory. Every command in the doc is copy-pasteable and every URL
    resolves. Anything that required improvisation gets folded back into the doc before the
    step is called done.

## Out of scope

- Fixing `cmd/uplink` (broken before this change; blocks its matrix row).
- Windows MSI installer and Authenticode signing (Storj has both; needs a Windows runner).
- macOS codesigning and notarization. Needed for a clean first-run experience on macOS (Gatekeeper
  quarantines a manually downloaded binary), but it needs an Apple Developer account and a signing
  identity we do not have yet. The macOS binaries still ship and still auto-update; `docs/RELEASE.md`
  carries the `xattr -d com.apple.quarantine` note until this is done. Track it as the immediate
  follow-up to this work.
- Migrating the live fleet or the coord VPS to registry pulls - the plan makes it possible,
  switching production over is a separate operational change.

---

# As built (2026-08-07)

Everything above is the plan **as approved on 2026-08-05** and is left unedited so the
decisions stay auditable. This section records where reality diverged. The accurate,
maintained description of the system is `docs/RELEASE.md` in the repo - prefer it over
anything above.

## Decisions that changed after approval

| approved | shipped | why |
|---|---|---|
| publish to ghcr.io + Docker Hub | **internal GitLab registry only** | user decision. `REGISTRY` defaults from `$CI_REGISTRY_IMAGE`; `GHCR_ORG`/`DOCKERHUB_ORG` default empty and fan out when set. Removed 4 secrets from the setup: the GitLab registry authenticates with variables every job already has |
| macOS via Zig, osxcross as fallback | **native macOS runner** | Zig ships libSystem but not Apple's frameworks. `99designs/go-keychain`, `rjeczalik/notify`, gopsutil and the Prometheus process collector need CoreFoundation, CoreServices, IOKit and `mach/mach_vm.h`, which exist only in the macOS SDK. `scripts/release/build-native.sh` handles darwin; job is skipped when `DEPIN_MACOS_RUNNER_TAG` is unset |
| linux/arm ships | **blocked upstream** | go-aioz `x/mint/simulation/genesis.go:24` does `r.Intn(22143943760)`, which overflows a 32-bit int. One-line upstream fix (`r.Int63n`). Marked `blocked` in `PLATFORMS`; arm64 is unaffected |
| 7 components | **9 components** | `relay` and `jobq` split out of the coord binary - see below |
| `docker-bake-ci.hcl` registry cache overlay | **not built** | single runner today, so the local BuildKit volume suffices. Needed the moment a second runner is added |

## Scope added after approval

**relay and jobq became their own components** (`cmd/relay`, `cmd/jobq`), removing the
`coord relay` / `coord jobq` subcommands entirely. This started as an identity question
and turned out to be a real defect: as subcommands they inherited coord's single root
`--identity-dir` flag, so an operator who did not override it ran a relay holding the
**coordinator's** certificate and therefore the coordinator's libp2p peer ID. Wrong on
every axis - a relay fleet needs one identity per host to be addressable, revocable and
attributable, and the relay is the most exposed process we run while coord's identity
signs order limits.

Each binary now defaults to its own directory (`<appdir>/relay/identity`,
`<appdir>/jobq/identity`), with `TestIdentityDirIsNotCoordinators` as a regression guard.
The peer-ID migration is order-sensitive and is written up in `docs/RELEASE.md` §7.

## Defects found and fixed while executing

- **`.dockerignore` had never been committed.** The allowlist `.gitignore` swallowed it, so
  a fresh clone built images with no ignore file - the multi-GB-context incident that file
  itself warns about.
- **`**/hashstore` in a `.dockerignore` eats the source package** `worker/pkg/hashstore`.
  Never glob a bare package name at any depth.
- **`pkg/utility/hardwareinfo` had no freebsd implementation** (linux/darwin/windows only),
  so freebsd failed on `undefined: GetMACAddress`. Added `hardware_freebsd.go`, avoiding
  `jaypipes/ghw` which is Linux and Windows only. Compile-verified only; never run on FreeBSD.
- **`worker/pkg/du/diskusage.go` was not portable.** `Statfs_t.Bavail` is unsigned on Linux
  but signed on FreeBSD and darwin, so the fields would not multiply. `blocksToBytes`
  normalises through `int64` and clamps negatives to zero rather than wrapping to ~16 EB free.
- **32-bit needs `force32bit`** alongside `purego`: curve25519-voi selects its 64-bit generic
  field implementation under `purego` while excluding the 64-bit types.
- **`cmd/uplink` is gone**, not merely broken as the plan assumed.

## Test-stage findings (the plan assumed `go test ./...` just worked)

Running the suite for the first time gave 98 packages ok and 3 failing, root-caused to:

1. `CREATE EXTENSION IF NOT EXISTS "uuid-ossp"` is **not concurrency-safe**; parallel packages
   migrating their own schemas against one database collide on `pg_extension_name_index`.
   CI now creates it once before any test runs.
2. Postgres default `max_connections=100` is too low for testplanet's coord + workers +
   ranged loop. The service now starts with `-c max_connections=500`.
3. The test job was missing `-tags purego`, so it exercised a different build of the crypto
   paths than the one that ships.

With those fixed, **one genuine failure remains**: `TestRepairRebuildsLostPiece` fails with
`repairer: no pieces uploaded` (`coord/repair/repairer/segments.go:394`). Proven pre-existing
- it fails identically with all of this branch's changes stashed. Because `test` is an earlier
stage than `build`, **it currently blocks every pipeline**, including merge requests.

## Verification status against the plan's own list

| item | status |
|---|---|
| 1 allowlist | done |
| 2 single-platform binaries | done - 9 binaries, statically linked, correctly stamped |
| 3 cross-compilation | done for linux/amd64, linux/arm64, windows/amd64, freebsd/amd64. Could not execute non-native binaries: no qemu binfmt on the workstation, so the static-link claim rests on `readelf -d` showing no dynamic section |
| 4 macOS | not run - needs the Mac runner |
| 5 images | coord, relay, jobq built and verified running as uid 10001. Multi-arch push not exercised |
| 6 end-to-end runtime | not run |
| 7 fleet regression | not run |
| 8 provenance | not run - cosign and syft are not installed on the workstation |
| 9 pipeline | not run - needs the runner registered |
| 10 updater | not run |
| 11 runbook | `docs/RELEASE.md` written; the dry run that validates it has not happened |

## Open items

1. `TestRepairRebuildsLostPiece` - fix, skip with a tracking issue, or `allow_failure: true`.
2. Register the Linux runner and the CI variables; run an MR pipeline.
3. Confirm the GitLab registry version handles OCI attestation manifests (`unknown/unknown`
   platform entries). If not, drop `attest` or accept unattested internal images.
4. `docker-bake-ci.hcl` registry cache, before a second runner exists.
5. Upstream `r.Int63n` fix in go-aioz to unblock linux/arm.
6. Relay/jobq identity migration on the live fleet, per `docs/RELEASE.md` §7.
