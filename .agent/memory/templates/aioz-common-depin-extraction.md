---
type: fact
tags: [aioz-common, depin, go, refactor, git-subtree]
created: 2026-09-08
agent: main
---

2026-09-08: twelve domain-free packages were moved out of `depin`
(`/home/tuan/work/depin-workspace/depin`, module `aioz-depin`) into
`aioz-common` on branch `feat/depin-utils-and-secret` (not pushed, not merged,
not tagged). depin itself is untouched and still does not import aioz-common.

Moved, keeping their original package names and flattened to top level:
`memory`, `date`, `period`, `slices2`, `context2`, `signal`, `leak`,
`lrucache` (from `internal/`), `time2`, `errs2`, `base58` (from
`pkg/common/`), `readcloser` (from `pkg/`). Names were deliberately not
changed, `2` suffixes included, so a later depin migration is an import-path
rewrite rather than an API rewrite - same rule as
[[aioz-common-consolidation]].

Mechanics that worked: `git subtree split -P <dir>` in depin prints a SHA
without creating a branch, but `git subtree add` needs a ref, so point a temp
branch at each SHA, `git subtree add --prefix=<name> <depin-path> <branch>`,
then delete the temp branches. depin is only 986 commits, so all twelve splits
take seconds.

HISTORY CAVEAT, more precise than the note in [[aioz-common-consolidation]]:
`git blame` resolves to the original depin commits, but `git log -- <path>`
stops at the import commit **and `--full-history` does not help**, because the
pre-import path was different (`internal/memory/size.go`, not
`memory/size.go`). Reach the earlier history through the import commit's second
parent: `git log $(git rev-list --parents -n1 <import-commit> | awk '{print $3}')`.

Required conversions on import:
- `github.com/zeebo/errs` (v1) -> `errs/v2` in `errs2` and `readcloser`, to
  avoid two major versions of the same library. `errs.New` -> `errs.Errorf`.
  `errs.IsFunc` does not exist in v2; `errs2.IsCanceled` is now plain
  `errors.Is`, which is equivalent because v2's grouped-error type implements
  both `Unwrap() []error` and `Is`.
- `lrucache`'s tests needed `Go`/`Wait`/`Cleanup` on a test context, which
  `stats/testcontext` lacked, so those were added there rather than importing
  depin's 336-LOC `internal/testcontext`.
- `misspell` in the shared lint config enforces US spelling, so imported
  British spellings (`cancelled`) fail the gate.

DELIBERATELY LEFT IN DEPIN: `pkg/checksum` and `pkg/merkle` (clean, but
storage-integrity primitives that belong beside the piece pipeline - user's
call). Also blocked: `sync2` and `pkcrypto` each exist as **two live diverged
forks** in depin (`internal/sync2` vs `pkg/common/sync2`,
`pkg/pkcrypto` vs `pkg/common/pkcrypto`) and must be merged first; eight more
packages (`pkg/lifecycle`, `pkg/debug`, `pkg/version`, `internal/testcontext`,
`pkg/testmonkit`, `internal/grpcerr`, `internal/cfgstruct`, `sync2.Cycle`)
duplicate something aioz-common already has and need API reconciliation, not a
move.

The full inventory (36 STRONG / 21 WEAK / domain-bound) lives in the plan at
`/home/tuan/personal/tuan/Projects/templates/plans/2026-09-08-aioz-common-extraction-and-secret-package.md`.
