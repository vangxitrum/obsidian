---
type: project
tags: [aioz-stream, refactor, vertical-slices, live, swagger]
created: 2026-09-10
agent: main
---

`internal/app/live` decomposed into nested sub-slices on branch
`refactor/sync-with-template-source` (worktree 2:
`/home/tuan/.treehouse/aioz-stream-370b08/2/aioz-stream`). Commits
`699f2cac` (decompose) and `e61abbe1` (lint). Continues
[[slice-migration-state]], which had left `live` as one flat 4086-line package.

```
internal/app/live/
  endpoint.go   composes only, no handlers
  stream/  media/  multicast/  statistic/
```
Each sub-slice is a full slice: `common.go` (Error tag `live.<name>`),
`store.go`, `dto.go`, service, endpoint with its own `Routes(group)`. The
parent builds the three echo groups (api-key, public player, webhook-token) and
hands each to the sub-slices, so auth and rate limiting are unchanged.
`stream_endpoint.go` (1979 lines) became `media/endpoint.go` (1266) +
`stream/endpoint.go` (518).

**The cycle rule that makes it work:** a sub-slice never imports a sibling or
the parent. It declares the narrow interface it needs next to its consumer -
`multicast.StreamKeyLookup` (1 method), `media.StreamKeyLookup` (2),
`media.StreamKeys` (1, satisfied structurally by `*stream.Service`). Parent
imports child, never the reverse. Same shape as the `Media` interface noted in
[[slice-migration-state]].

**DTO convention:** user asked for request+response merged into one `dto.go` per
package. Tree now has both forms - `request.go`/`response.go` in 11 slices,
`dto.go` in 9 packages. `dto.go` is the form for new packages; do not churn the
existing pairs.

**Two swag traps, both cost a full debug cycle:**
1. A wire type with no `//	@name` gets its schema key from the *package path*,
   so moving it silently renames the model in the spec and every SDK.
   `LiveStreamMulticast` had none; added one. Check `@name` before moving any
   wire type.
2. swag resolves an annotation's type through *that file's own import set*. A
   file referencing `domain.ResponseError` only in `@Failure` lines fails with
   "cannot find type definition" once it stops importing `domain`. Fully
   qualifying the path does not work - swag parses the `10.0.0.50` module path
   as the package name. Fix: import it and anchor with `var _ domain.ResponseError`.

**`internal/tools` guardrails break under CI's GOMODCACHE.** `.gitlab-ci.yml`
sets `GOMODCACHE: $CI_PROJECT_DIR/.cache/go-mod`, so the repo-walking tests
(`TestEmbeddedAssetsAreNotGitignored`, `TestNoGoSourceIsGitignored`) scan every
dependency's own `.gitignore` and `//go:embed` and report ~70 third-party
findings in 96s. Fixed with a shared `isVendored()` skipping
`vendor/ .cache/ mediamtx/ w3streamcore/` plus `go env GOMODCACHE`.

**`.gitignore` is an allowlist** ([[repo-layout]]): a new root file is invisible
to git until named. `SOURCE_STRUCTURE.md` needed `!/SOURCE_STRUCTURE.md`. Written
at repo root, modelled on `depin/SOURCE_STRUCTURE.md`, and documents the slice
file convention, the dependency rules, the narrow-interface pattern and the swag
traps above.
