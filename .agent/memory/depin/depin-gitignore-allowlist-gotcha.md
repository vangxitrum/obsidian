---
type: fact
tags: [gitignore, build, gotcha, ci]
created: 2026-07-10
agent: main
---

This repo's `.gitignore` is **allowlist-style**: line 9 is a bare `*` (ignore everything), then specific `!pattern` exceptions allowlist back in what's tracked (`!*.go`, `!*.sql`, `!*.md`, `!docs/**`, etc., `.gitignore:1-52`). Any new file extension or path that needs to be committed — especially non-`.go` assets a Go file depends on via `//go:embed` or the toolchain's implicit assembly-file convention (`*.s` alongside `*.go` in the same package) — silently fails to commit unless explicitly allowlisted, with no error at commit time. The break only surfaces later as a `go build` failure on a **fresh clone** (developers with an existing local checkout have the file as an untracked-but-present local artifact and never notice).

Hit 3 times on branch `feat/coord-metrics-collection` ([[coord-metrics-collection-shipped]]):
1. `coord/relay/status.html` (`//go:embed status.html` in `coord/relay/status.go`) — never committed, broke `go build ./...` on this treehouse worktree (a genuinely fresh clone) even though `origin/main`'s `f4b925d` "built fine" for everyone with an existing checkout.
2. `pkg/infectious/addmul_amd64.s` — required by the Go toolchain to satisfy `//go:noescape` stubs in `addmul_amd64.go`, same failure mode.
3. `monitoring/grafana/dashboards/*.json` (new in this branch, Part B) — needed `!monitoring/**` added before the dashboard JSON could be committed at all.

**How to apply:** before trusting `go build ./...` "works" on this repo, verify on an actual fresh clone (or `git stash` + a scratch clone), not just the existing dev checkout. When adding any new non-`.go`/`.sql`/`.md`/`.proto`-family file that must be committed, check `git status --short` after `git add` — if it doesn't show up as staged, check `git check-ignore -v <path>` and add the narrowest possible `!pattern` exception near the other specific carve-outs in `.gitignore` (not a broad wildcard).
