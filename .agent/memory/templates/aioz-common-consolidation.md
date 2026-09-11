# aioz-common consolidation

2026-09-08: `aioz-config`, `aioz-logger` and `aioz-stats` were merged into one Go
module, `gitlab.internal/tuan.quang.tran/aioz-common`, cloned at
`/home/tuan/work/templates/aioz-common`. `aioz-template` stays a separate repo -
it is a service scaffold, not a library.

Layout: `config/`, `logger/`, `stats/{debug,version,testcontext}`. Package names
were deliberately NOT renamed, so `version` and `testcontext` still sit under
`stats/` even though they are not stats - flattening would have broken import
paths for no functional gain.

Import path map:
- `aioz-config` -> `aioz-common/config`
- `aioz-logger` -> `aioz-common/logger`
- `aioz-stats/debug|version|testcontext` -> `aioz-common/stats/...`

History was imported with `git merge -s ours --allow-unrelated-histories` plus
`git read-tree --prefix=`. Consequence: `git blame` resolves to the original
commits correctly, but `git log -- <path>` stops at the import commit unless you
pass `--full-history`. Pre-merge paths are still reachable (e.g.
`git log --full-history -- handler.go`).

`zeebo/structs` must stay pinned at `v1.0.3-0.20230601144555-f2db46069602`;
`go mod tidy` on a fresh go.mod silently resolves it down to `v1.0.2`.

`aioz-logger` had no `.golangci.yml` or CI of its own and failed the shared lint
gate on import (2 unchecked `fmt.Fprintln`, 1 gofmt violation) - fixed in the
unification commit.

Second pass, same day: `apierr`, `cycle`, `lifecycle`, `logging` and `monitor`
were moved out of `aioz-template/internal/utils` into aioz-common (`monitor` to
`stats/monitor/`), carried over with `git subtree split` per directory.

`stats/version` absorbed the template's `version` package - the plain string
vars that `-ldflags -X` patches, the dev/release defaults channel, `Register`
and `String`. `Info()` had to be renamed `Current()`: it collided with the
`Info` type once the two packages merged.

SUPERSEDED 2026-09-08 by commit a01f944 ("derive the build stamp from Go build
info"): `Register` and `Current` no longer exist. The package's whole exported
surface is now `Info`, the `Build` package var filled from
`debug.ReadBuildInfo()` in `init()`, `IsRelease()` and `Channel()`. See
[[aioz-common-version-build-stamp]]. Any service's Makefile must now aim
`-X` at `aioz-common/stats/version.{release,sha,build,channel}`.

`logging` stays a separate package from `logger` on purpose - both declare
`Config`, `FileConfig` and `New`, so folding them together is an API rewrite.

Deliberately left in aioz-template: `migrate` (would drag golang-migrate + pgx
into every consumer's module graph) and `naming` (a `_test` package that walks
its own module enforcing playbook rules).

Note: `apierr/policy_test.go` walks up to the enclosing `go.mod`, so it now
polices aioz-common rather than aioz-template - the template lost that
repo-wide reason-slug check.

Third pass: `stats/version` was rewritten to storj's model. `Build` is filled
in `init()` from `debug.ReadBuildInfo()` - `vcs.revision`, `vcs.time`,
`vcs.modified` - which Go embeds automatically since 1.18, so a service needs
NO ldflags block for a correct `/version/`. The `-X` vars (`buildVersion`,
`buildCommitHash`, `buildTimestamp`) are now fallback only, consulted when the
VCS block is missing: a container build whose context has no `.git`,
`-buildvcs=false`, and always inside `go test`. `buildRelease` is the one that
overrides rather than fills a gap.

`Info.Release` is DERIVED, not declared: true only if version + commit +
timestamp are set and the tree was unmodified. Distinct from `channel`, which
stays an explicit `-X` var because it picks devDefault vs releaseDefault config
tags - a choice, not an observation. `Register()` and `Current()` are gone.

`Info` fields are now Version/CommitHash/Timestamp/Release/Modified.

Go pseudo-versions are treated as "no version". Three forms exist; the
timestamp separator is `-` in `vX.0.0-<ts>-<hash>` but `.` in
`v1.0.3-0.<ts>-<hash>`. A naive "dash then 14 digits" check misses the latter -
match the suffix instead.

`apierr` was trimmed: `reason.go` (ValidReason/statusWords), `policy_test.go`
and `readme_test.go` deleted - they enforced one project's naming rule on one
repo. `apierr.go` + `trace.go` stay, so services define their own constructors
over `*apierr.Error`.

golangci-lint here runs `misspell` with US spelling - British forms
(synthesises, recognised, capitalising) fail the build in .go files.

Fourth pass: added `uuid/` and `currency/`, adapted from
`depin/internal/vo` (which is where these value objects live - `UUID.go`,
`aioz_coin.go`, `usd.go`, `decimal.go`, plus depin-only protocol types).

`uuid` is `[16]byte`, NOT depin's `[]byte`. The slice form is not comparable,
cannot be a map key, and depin's `IDNil = UUID(uuid.Nil[:])` aliases
google/uuid's own `Nil` var - writing through a copy corrupts it process-wide.
Ids are v7 so they sort by creation time. gogo/protobuf methods kept but
implemented as byte copies, so no protobuf import.

`currency` uses a phantom type param: `Amount[U Unit]`, with `USD = Amount[USDUnit]`
and `AIOZ = Amount[AIOZUnit]` as generic aliases. Adding USD to AIOZ is a
COMPILE error. Verified.

KEY DEPENDENCY FACT: depin's `AiozCoin` is `types.Coins` (cosmos multi-coin
slice) wrapped to enforce single-token logic, and that costs
`github.com/cosmos/cosmos-sdk/types` = 638 packages / 73 modules. Modeled like
depin's own `USD` (an `sdkmath.Int` count of attoaioz) it needs only
`cosmossdk.io/math` = 2 modules. Whole aioz-common is now 24 modules.

golangci-lint staticcheck ST1020: a group-header comment (`// Arithmetic.`)
directly above an exported method is read as its doc comment and fails. Put a
blank line after the header.

`errs.Tag` (zeebo/errs/v2) has `Errorf` and `Wrap` - NOT `New`.

Still open: `aioz-template` still imports every old path, still has
`internal/utils/version`, and its Makefile LDFLAGS still target the removed
symbols. Nothing is pushed yet; `v0.1.0` is tagged locally on HEAD.
