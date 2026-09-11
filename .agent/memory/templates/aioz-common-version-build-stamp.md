---
type: decision
tags: [aioz-common, version, semver, monkit, go]
created: 2026-09-08
agent: main
---

2026-09-08, branch `feat/depin-utils-and-secret`: `stats/version` moved to
top-level `version/` and gained `SemVer` + a monkit `StatSource`, ported from
Storj's version package (depin's copy is `pkg/version`, 956 LOC).
`stats/version` remains as type aliases with `Deprecated:` notes.

SCOPE DECISION (user's): take the core only. Deliberately NOT ported: the
staged rollout (`RolloutBytes`, `PercentageToCursorF`, the
`HMAC-SHA256(seed,nodeID) <= cursor` candidate check) and `ShouldUpdateVersion`.
Those only mean something with a versioncontrol server + download URL +
checksum + updater binary, all of which live in depin with one consumer.
depin's `versioncontrol/` is 1909 LOC and its `internal/version/checker`
client imports `worker/pkg/p2pc` and `internal/vo`, so it is domain-bound
anyway.

`blang/semver/v4` added: exactly ONE module, its own go.mod has no requires.

`Info.Version` is now `SemVer`, not `string`. Two bugs in Storj's original
were fixed in the port:
- `String()` had a POINTER receiver, so a `SemVer` in a struct field never
  rendered through fmt. Also, the embedded `semver.Version` has its own
  `String()` (no `v` prefix) and `MarshalJSON` (uses that String), both of
  which get PROMOTED - so Storj's JSON says `2.4.1` while its String() says
  `v2.4.1`. Both must be shadowed on a value receiver.
- `IsZero()` used `reflect.ValueOf(sem).IsZero()`; replaced with field checks.

THE ORDERING TRAP, worth keeping: `semver.ParseTolerant` accepts
`git describe --tags` output like `v2.4.1-3-gabc1234`, but semver reads
everything after the dash as a PRERELEASE, so it sorts BELOW `v2.4.1` - i.e. a
binary built 3 commits past the tag believes it is older than the tag. Never
pass git-describe output as a build version; use `--abbrev=0`. There is a test
(`TestGitDescribeOutputSortsBelowItsOwnTag`) pinning this.

Storj's answer, adopted: an untagged build derives
`v0.0.0-dev.<unix>.g<short commit>`. Anchoring at `v0.0.0` with `dev` as a
prerelease makes every derived version sort below every real tag including
`v0.0.0`. The timestamp is a NUMERIC semver identifier, which compares
numerically not lexically, so derived versions order by build time. The commit
gets a `g` prefix because a semver identifier may not be all-numeric with a
leading zero. Our version derives this automatically from build info, so an
internal service gets a correctly-ordering version with NO Makefile at all -
depin's requires ldflags.

monkit encoding facts (Prometheus stores only float64): version as three
fields major/minor/patch; commit hash as `crc32` (changes exactly when the
commit does, so distinct-value count = number of live builds); `age_sec` shows
a stale instance without the query knowing the current version; os/arch as a
FIELD NAMED FOR ITS VALUE set to 1 (`os_linux`), so queries can group by it.
Storj memoizes the commit CRC on the `Info`; that cache is worthless (40 bytes
of CRC32 per scrape) and, if moved off the Info to a package global, silently
returns the first hash's value for every later Info - a test caught exactly
that.

`monkit.Registry` has NO `Add`. A `StatSource` attaches via
`registry.ScopeNamed("").Chain(src)`. `version.Register(nil)` wraps that
(nil = `monkit.Default`), mirroring `stats/monitor.Register`, including the
per-registry idempotence guard.

Verified end to end: a separate consumer module built with
`-ldflags "-X <pkg>.buildVersion=v2.4.1"` renders
`version_info{scope="",field="major"} 2` through the real
`debug.PrometheusEndpoint`.
