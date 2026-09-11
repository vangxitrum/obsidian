---
type: fact
tags: [depin, ci, docker, buildkit, release, gotcha]
created: 2026-08-26
agent: main
---

`release.Dockerfile`'s `build-binaries` stage failed in CI with
`no secure protocol found for repository` while downloading
`10.0.0.50/aioz-network/ethermint.git`, even though the `download-dependencies`
stage's `go mod download` had succeeded (and a standalone probe of it passed).

**The trap: a BuildKit `--mount=type=secret` is NOT inherited by child stages,
but the side effects of a RUN are.** `download-dependencies` writes
`/root/.gitconfig` (the `insteadOf` ssh->https rules) into its layer, so
`build-binaries` (`FROM download-dependencies`) inherits the URL rewriting - and
inherits *none* of the credentials, because the netrc only ever existed as an
ephemeral tmpfs mount. Half the setup survives, which is why it looks like it
should work.

**Why it only fires sometimes:** `build-binaries` normally never touches the
network, because `/go/pkg/mod` is warm. But the layer cache and the cache mount
have independent lifetimes: recreating the buildx builder (the
`docker buildx rm` / `create` dance in the CI job) drops cache-mount state while
the registry layer cache survives. Result: `download-dependencies` is a layer
CACHED hit so `go mod download` never re-runs, the mod cache is empty, and
`go build` has to fetch the private module graph itself - unauthenticated.

**Fix** (2026-08-26, branch `docs/update`): add
`--mount=type=secret,id=netrc,target=/root/.netrc` to the `build-binaries` RUN.
The bake target already declares the secret (`release.docker-bake.hcl` `_common`),
so no HCL or `bake.sh` change was needed.

**How to diagnose this class:** build a probe stage twice, with and without the
secret mount, and print `test -f /root/.gitconfig` / `test -f /root/.netrc`
before the go command. Without the mount: gitconfig YES, netrc NO, and git dies
with `could not read Username ... terminal prompts disabled`. With the mount
(even a bogus token): netrc YES and the error changes to a server-side 404/403,
proving the credential path is actually taken. Force an empty module cache with
`GOMODCACHE=/go/probe-mod` plus a `BUST` build-arg rather than trusting
`--no-cache`.

Related: [[docker-bake-release-pipeline]], [[ci-release-drift-fixes]],
[[component-version-tags]]
