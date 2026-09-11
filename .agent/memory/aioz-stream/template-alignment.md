---
type: decision
tags: [aioz-template, apierr, security, ci, lint, aioz-stream]
created: 2026-09-07
agent: main
---

Aligned `aioz-stream` with the house scaffold at `~/work/templates/aioz-template`
on branch `refactor/sync-with-template-source` (branched at `10b1f8ea`, previously
empty). Scope chosen by the user: **contracts first, no file moves**. The
controllers/services/repositories layout stays; what changed is the error
contract, the guardrail tests, and the build/CI surface.

**Why not a full restructure:** the template is a greenfield scaffold (cobra +
aioz-config/aioz-logger/aioz-stats, vertical slices under `internal/app/<slice>`,
a `Peer` lifecycle). aioz-stream is ~200 files with viper env config and a
795-line `func init()` composition root. A full port touches every file of a live
service; the enforceable contracts are the part that pays off immediately.

**What landed**
- `internal/utils/apierr/` ported verbatim from the template (module path
  rewritten, `github.com/zeebo/errs/v2` added). Its `policy_test.go` walks the
  whole module - skip list extended with `mediamtx`, `w3streamcore`, `proto`,
  `trace_data`.
- One error renderer: `response.renderError` backs both `ResponseError` and the
  new `response.HTTPErrorHandler`, registered on the echo server in
  `cmd/http/init.go`. Before this there was **no** custom `HTTPErrorHandler`, so
  any middleware returning a raw `*echo.HTTPError` answered in echo's own
  `{"message":...}` envelope.
- `GeneralResponse` gained `reason` and `request_id` (both `omitempty`, so
  existing clients see no change). 5xx bodies now carry the reason slug and a
  fixed message; the cause, `errs` tags and origin stack go to the log.
- `humanize` ported from the template's `server/message.go` - every client
  message is capitalized and full-stopped at the one point it leaves the service.
- Guardrails added: `apierr/{policy,apierr}_test.go`,
  `response/{message,diagnostics}_test.go`, `docs/spec_test.go`,
  `version/version_test.go`, `config/config_test.go`.
- `internal/utils/version` (no aioz-stats dep) + `/version` route + `make
  verify-stamp`, which greps the built binary for the sha because `-X` against
  anything but a plain string var is dropped silently.
- Makefile: `-tags purego` on build (it was missing while `.golangci.yml` and CI
  both claimed it was there), `-trimpath`, ldflags, `go test -race`, and
  non-interactive `swagger` / `swagger-verify`.
- `.gitlab-ci.yml`: new `test` stage between lint and build running
  `make swagger-verify` + `make test`, `CGO_ENABLED=1` for -race.

**Gotchas hit**
- `SWAG_VERSION` must be **v1.16.6**. v1.16.3 rewrites the whole spec (712-line
  diff); v1.16.6 reproduces the committed one exactly. The committed spec *was*
  stale by one field (`source_url` from the HLS-import feature).
- `docs/spec_baseline.json` baselines all 100 operations: 57 have no `@ID` at
  all and 43 have an `@ID` that is not lowerCamel, so every SDK method name today
  is either invented from the URL or non-idiomatic. The baseline only shrinks -
  the test fails if an entry no longer needs to be there.
- The global Go formatter hook re-wraps whole files on `Edit`; use Bash/python
  edits on large files or the diff explodes (confirmed again on
  `cmd/http/init.go`: 36 insertions vs the 4 actually intended).
- `golangci-lint run ./...` had **9 pre-existing findings** on this branch
  despite `allow_failure: false` (md5 + 0755 dirs in the newer highlight code,
  one goimports). Extended the existing documented gosec exclusions to cover the
  new sites; now 0 issues.

**How to apply:** new code returns `apierr` from the service and lets the
endpoint pass it through - never pick a status in a handler. Reason slugs name
the subject (`media-not-found`), never the status; `policy_test.go` enforces it
across the module.

See [[aioz-stream-bug-report-2026-09]] for the defects found along the way.


**The template moved to aioz-common, and so has this repo (2026-09-08).**
`~/work/templates/aioz-template` now depends on
`gitlab.internal/tuan.quang.tran/aioz-common` instead of carrying its own
copies ("depend on aioz-common v0.1.0 instead of three libraries and five
copies"). aioz-stream had copied those very packages from the template, so
they were duplicates of a duplicate.

Swapped to `aioz-common v0.2.0`:
- `internal/utils/apierr` -> `aioz-common/apierr`
- `internal/utils/cycle` -> `aioz-common/cycle`
- `internal/utils/lifecycle` -> `aioz-common/lifecycle`
- `internal/utils/version` -> `aioz-common/stats/version`

The first three had **byte-identical exported APIs**, so those were a pure
import rewrite across 54 files. Check with
`diff <(grep '^func ' ours) <(grep '^func ' theirs)` before assuming it.

**`version` was not a drop-in.** aioz-common's reads the commit, its timestamp
and the dirty flag from **Go build information** (`debug.ReadBuildInfo`, which
`go build` records by itself since 1.18), with link-time vars only as a
fallback. So its symbols are `buildVersion`/`channel`, not
`release`/`sha`/`build`/`channel`, and the accessor is `version.Build` rather
than `version.Get()`. The `-X` flags carrying sha and build are gone, and CI no
longer passes `SHA=` - it was duplicating what the toolchain already embeds.
`verify-stamp` now greps for RELEASE, since that is the only value an ldflag
still sets.

**The reason-slug policy stays local**, in `internal/utils/policy` - the
template does the same. aioz-common's apierr was reduced to its core and the
slug rules were trimmed out of it, which is arguably where they belonged: a
policy test walks a module with go/parser and therefore only ever covers the
module it ships in, so a shared copy could never have policed this one.

Still local and still ours: `internal/utils/migrate` (aioz-common has no
migrate), `internal/utils/chore`, `internal/utils/schedule`,
`internal/utils/log`. aioz-common also ships `config`, `logger`, `uuid`,
`currency` and `stats/debug` - none adopted yet; `config` in particular would
replace the viper loader and is a separate change.


**aioz-common v0.3.0 does not exist yet (checked 2026-09-08).** The newest tag
is v0.2.0; `git ls-remote --tags` confirms it. v0.3.0 is the **unreleased** work
on the default branch - its CHANGELOG says so - carrying twelve packages
imported from depin plus `secret`, `jwt` and the move of `stats/version` to
`version`. To use any of it the module has to be pinned to a pseudo-version
(`v0.2.1-0.20260908094255-93c696c1e41d`), which is an untagged commit and can
be force-pushed away.

**`jwt` adopted.** `internal/utils/token` is built on `aioz-common/jwt`:
algorithm pinned at construction, the header's `alg` never trusted. The public
API (`CreateCredential`, `ValidateAccessToken`, `ValidateRefreshToken` returning
a map) is unchanged, so no caller moved. Two traps:
- **`secret.Token.String()` returns "[redacted]"** - that is the type's whole
  purpose. `Reveal()` is the value. Returning `String()` would have handed every
  user the literal string "[redacted]" as their access token; only the
  round-trip test caught it.
- **Switching the library changes the token format.** The old issuer nested the
  subject as a map under `sub`; `jwt.Claims.Subject` is a string. A legacy token
  therefore *passes* signature and expiry checks and yields an empty subject, so
  `verify` treats an empty user_id as a miss and falls through to a legacy
  reader. Without that every user is logged out at deploy.

**`uuid` and `currency` were measured and deliberately NOT adopted** - both are
data migrations, not type swaps:
- `uuid`: 1016 references across 181 files, and `uuid.UUID.Value()` returns 16
  raw bytes where `google/uuid` returns the string form, against live `uuid`
  columns.
- `currency`: worse. `Amount.GormDataType()` is `NUMERIC(60,0)` but our money
  columns are **`text`** (what AutoMigrate made of `decimal.Decimal`), `Value()`
  writes the **base-unit** count, and `FromString` parses base units - so an
  existing `"1.5"` is either rejected or read as 1.5 attoaioz. Swapping the type
  silently misprices every account by 10^18. It also changes the JSON money
  format for every SDK client.
Both need their own change with a column migration and an API-version story.


**uuid migrated to aioz-common (2026-09-09).** 183 files, 1016 references.
`google/uuid` survives only in the probe test, which compares the two.

**The migration was gated on a database probe, not on reading the code.**
`uuid.UUID.Value()` returns 16 raw bytes where google/uuid returns the string
form, so the question was whether pgx accepts that for a `uuid` column.
`internal/tools/uuidprobe` (build tag `uuidprobe`) proves three things against a
real Postgres: a round trip works, the stored `id::text` equals `id.String()`,
and a row written the old way by google/uuid reads back through the new type.
Existing data is therefore untouched.

Bonus: the generated ids are **version 7** - time-ordered, so they cluster in
an index instead of scattering like v4.

Call-site mapping: `New()` -> `Must()`, `NewString()` -> `Must().String()`,
`Parse` -> `FromString`, `Nil` unchanged, and the type keeps the name `uuid.UUID`
so only the import path moves. **`MustParse` has no equivalent** - a parse that
cannot fail is a test-only need - so affected tests got a local `mustUUID`
helper. External SDKs still speak google/uuid: convert at the boundary with
`.Google()` and `uuid.FromGoogle()`, as `internal/utils/payment` now does.

**Naming: Controller is gone.** `MediaController`, `MediaCaptionController`,
`MediaChapterController` and `LiveStreamController` became `*Endpoint`. The
slice's route registrar keeps the bare `Endpoint` name every other slice uses,
so the composition root's variables are `mediaEndpoint` (registrar) and
`mediaHandler` / `mediaCaptionHandler` / `mediaChapterHandler` (the handler
holders) - renaming both to `mediaEndpoint` collided.

**Two more hand-rolled loops became chores**: `media:watch-playlist` (was
`StartWatchPlaylist`, a 5s ticker plus a goroutine plus a stop channel closed
by `StopCron`) and `media:hls-import` (was `go hlsImportService.Run(ctx)`,
which owned its own ticker). Both are now a `RunOnce`/`Pass` method with the
schedule owned by `chore`. `StopCron` is a documented no-op; the other two
channels it closed belonged to watchers commented out in the composition root.

Still started with a bare call: `paymentClient.StartWatchingPayment(ctx)` -
the last one.
