---
type: decision
tags: [aioz-stream, refactor, models, store, layering]
created: 2026-09-07
agent: main
---

`internal/models` was a 59-file grab bag: entities, 38 repository interfaces,
HTTP DTOs and a 1096-line `variable.go`. Split by layer, 2026-09-07, closing the
slice migration ([[slice-migration-state]]).

**Where things went**
- **Repository interfaces -> `internal/store`**, one file per former models
  file, plus `store.UseCase`. `pkg/v1/repositories` implements it. This
  supersedes deviation 1 recorded in [[slice-migration-state]]: repositories are
  still shared, but the interfaces no longer live beside the entities.
- **DTOs -> the slice that serves them.** A type referenced by `internal/store`
  is a store shape and stays in models (`GetMediaListInput`,
  `GetAggregatedMetricsInput`, ...); a type only one slice touches moves into
  it. 7 DTOs were referenced nowhere and were deleted.
- **`variable.go` -> 11 per-domain files**: `variable_{app,status,media,limits,
  sort,language,analytics,billing,report,account,mediamtx}.go`.
- `internal/controllers` dissolved: `CallerAuth`/`CallerId` -> `internal/middlewares`
  (the package that puts `authInfo` in the context), `AllowedValues` ->
  `internal/utils/response` (beside `humanize`). `internal/controllers/dto` split
  into the editor and highlight slices; editor imports highlight for the media
  response it embeds, which is the one place a slice imports a sibling.
- `pkg/v1/services` is gone. `AuthService` and `CdnUsageService` had zero
  methods and were dead wiring; `MailService` became `internal/app/mail`.

**How to do a move like this: let the compiler drive it, not a type list.**
Extracting the interfaces with a hand-written list of names to prefix with
`models.` missed a dozen types and dropped imports. What worked was a loop:
build, parse `undefined: X` out of the error, prefix `models.` (or add the
import if X is a package), repeat. 3-4 passes converge.

Two traps in that loop:
- A **syntax error masks every type error**, so the loop reports "clean" while
  the package still does not build. Check the build output, not just the
  absence of `undefined:`.
- A blind `\bX\b -> models.X` also rewrites **field names and struct-literal
  keys** (`Controls *Controls` -> `models.Controls models.Controls`). Anchor the
  regex to type position, or fix the handful of collisions by hand afterwards.

Cutting a `type X struct {...}` block with a regex ending at `^\}` **overshoots
when the struct ends with a trailing comment** - `} //\t@name\tX` is not `}` -
and swallows the next declaration. Three types moved slices by accident this
way; check the moved file's top-level declarations against what you intended.

`goimports` guesses wrong on ambiguous package names: it resolved `uuid` to
`storj.io/common/uuid`. Run it with `-local 10.0.0.50/tuan.quang.tran/vms-v2`
to match `.golangci.yml`, and check what it added.

**Request correlation, added the same day.** The id the service logs and traces
under is now sent back as the `X-Request-Id` response header, set by
`middlewares.AddLogContext` **before** the handler runs - so it is on a
manifest, an mp4 and a thumbnail too, none of which have a JSON envelope to
carry `request_id`. Inbound, `X-Request-Id` is read first and the old
`stream-trace-id` is still accepted. `response.RequestId(ctx)` is the one
lookup, used by both `ResponseSuccess` and `renderError`, so the header and the
body can never disagree. CORS needed the header in **both** `AllowHeaders` and
`ExposeHeaders` - without the latter a browser cannot read it at all.

The id still names a file under the trace directory, so it keeps the
`^[A-Za-z0-9_-]{1,64}$` restriction and an unsafe value is replaced rather than
echoed. `AddLogContext` stays on the slices group rather than `server.Use()`:
global would give 404s an id too, and every 404 would then write a trace file,
which is a free way for a scanner to fill the disk.

**Slice-owned entities (`internal/app/<slice>/model.go`), and why only three.**
The vertical-slice endgame is an entity plus its store contract living in the
slice that owns them, with `pkg/v1/repositories` implementing the slice's
interface. Measuring the reference graph first said only a handful qualify. A
models file can move into slice X only if:
1. no other models file references it (the media core is a 9-file cycle:
   `video, quality, caption, chapter, format, playlist, stream, variable_media,
   video_watermark` - none of them can ever move alone), and
2. the store file declaring its repository serves only that entity (otherwise
   `internal/store` would have to import the slice, and the slice imports
   store), and
3. nothing the slice transitively imports pulls in `pkg/v1/repositories`,
   because repositories must import the slice to implement its contract.

Landed: `ip` -> statistic, `email_connection` -> user, `variable_report` ->
report. Blocked, with the specific reason: `highlight_chunk`/`highlight_clips`
(highlight_media references them), `part` (video.go), `session` (action.go),
`wallet_connection`/`exclusive_code` (user.go), `watermark`
(video_watermark.go), `live_stream_key` (live_stream_video.go),
`live_stream_multicast` (store/live_stream_key.go), `subcribe`
(store/user.go), `watch_info` (store/session.go), `live_stream_statistic`
(live -> usage -> repositories -> live).

**Rule 3 is the one that bites, and it is a layering violation worth fixing on
sight.** Four slices and `internal/utils/payment` were calling
`repositories.MustNew*Repository(tx, false)` inline to rebind a repo to a
transaction, which quietly put every one of them downstream of the
implementations. `payment` is fixed - it now takes a `RepoFactory` of
constructor funcs from the composition root. `media`, `playlist`, `usage` and
`highlight` still do it in `service.go`, and until they stop, no entity can
move into them.

**`internal/app/video` is now `internal/app/media`** - the slice handles audio
as well as video, and only genuinely video-specific names (`VideoConfig`,
`VideoCodec`, `haveVideo`) kept the old word.

Splitting `internal/models` into per-domain **packages** was measured and is
legal - the domain graph is acyclic once `cdn_file` and `variable_billing` sit
in a base package and `webhook` is its own - but it rewrites 3172 references
across 200 files and 5 of the 9 natural names collide with slice packages.
Not done; recorded here so the measurement does not have to be redone.

**The layering inversion that unblocked the pattern.** The consumer-declared
interface is the fix: each slice declares the subset of repository constructors
it needs as its own `RepoFactory` interface, next to the use, and the
composition root passes one `repositories.Factory{}` value that satisfies all
of them structurally. `pkg/v1/repositories/factory.go` is the single
implementation - one method per repository, each returning the `store` (or
slice-owned) interface.

Before this, `media`, `playlist`, `usage`, `highlight` and
`internal/utils/payment` all called `repositories.MustNew*Repository(tx, false)`
inline to rebind a repo to a transaction. That put them downstream of the
implementations, so `pkg/v1/repositories` could never import a slice - which it
must do to implement a slice-owned contract. No production package under
`internal/` imports `pkg/v1/repositories` any more except `internal/seeds`.

Entities now owned by their slice: `ip` + `email_connection` +
`variable_report` (before the inversion), then `live_stream_statistic` and
`media_import`. Each is `internal/app/<slice>/model.go` holding the entity and
its repository interface, implemented by `pkg/v1/repositories`.

**What still blocks the rest, in order of how often it bites**
1. *A shared entity in models references it* - `part` (video.go), `session`
   (action.go), `watermark` (video_watermark.go), `wallet_connection` and
   `exclusive_code` (user.go), `live_stream_key` (live_stream_video.go). The
   9-file media cycle can never move at all.
2. *Its repository interface is declared in a store file that serves a shared
   entity* - `subcribe` (store/user.go), `watch_info` (store/session.go),
   `live_stream_multicast` (store/live_stream_key.go).
3. *Moving it creates a slice-to-slice cycle* - `video_usage` -> usage and
   `report_content` -> report both fail because `media` already depends on
   those slices. `api_key` -> apikey fails the same way through
   `internal/middlewares`.

Reason 3 is the interesting one: it is the "a slice never imports a sibling"
rule showing up as a hard compile error rather than a convention. An entity can
only move into a slice that nothing else already depends on.

**Automating a move like this: build after every single one, and roll back.**
A cycle is only visible at compile time, so the loop is move -> goimports ->
build -> keep or restore from a snapshot. Two script bugs to expect: stripping
`models.` from the whole moved file (it also unqualifies the types that stayed
behind), and forgetting that the moved file's *own* unqualified references to
things still in models now need the prefix added (`PendingStatus`,
`ExpiredEmailTime`). And in zsh, `set -- $pair` inside a loop does not
word-split, so a shell-driven sweep silently no-ops and reports success -
drive it from python.

**`internal/store` is now the common package - for shared contracts only.**
The right reading of "have a common package" was not another home for shared
entities (`internal/models` already is that) but shrinking `internal/store` to
the contracts two or more consumers use. It went from 35 files to 19; 14
single-consumer contracts moved into `internal/app/<slice>/store.go`
(media x6, live x2, user x2, playlist, report, statistic).

**Moving a contract is safe; moving an entity usually is not.** A contract
moving out of `internal/store` creates no `store -> slice` edge at all -
`pkg/v1/repositories` imports the slice to implement it, and nothing under
`internal/` imports repositories any more. Moving an *entity* rewrites
`models.X` to `slice.X` everywhere including inside `internal/store`, and the
moment `store` imports one slice, every slice that reaches `store` cycles.
That, not the entity's location, is what blocked the earlier attempts.

**Two things gate a contract move:**
1. Nothing else *inside* `internal/store` may reference it. `store.UseCase`
   aggregates User + ApiKey + Webhook, which is why those two cannot move
   despite having one consumer each. Count intra-package references, not just
   `store.X` call sites.
2. `pkg/v1/repositories` must not import anything the slice's own **test**
   binary reaches. `internal/app/highlight`'s integration tests build real
   repositories in-package (they call the unexported `generateThumbnails`, so
   they cannot become an external test package), which means repositories may
   import neither `highlight` nor `editor` - editor imports highlight for the
   media response it embeds. Both were moved and then reverted for this.

**`internal/middlewares` was split**: `auth_middleware.go` -> a new
`internal/middlewares/auth` package. Ten slices import `middlewares` and nine
of them only want `NewRateLimiter`, but the auth middleware in the same package
pulled in `internal/store`, so every slice transitively depended on store. The
remaining `middlewares` (rate limiter, log context, caller, admin key) depends
only on `models`.

**Why `internal/models` still exists, measured.** Every one of its 50 files is
used by two or more consumers - there is not a single one left with one owner
that could move into a slice. `video` is referenced by 14 packages, `user` and
`authentication_info` by 15, `error` and `variable_language` by 13. That is the
definition of a shared vocabulary, and it is what a common package is for. The
folder is not leftover work; it is the residue after everything with a single
owner had already left.

Two apparent zero-consumer files were checked: `video_thumbnail.go` was an
empty file (`package models` and nothing else) and was deleted; `variable_media`
reads as unused only because the measurement matched its one `type` and ignored
its heavily used constants.

**File convention: the main type leads.** After imports comes the primary type
(and any interface), then constructors, then constants and package vars. 18
files had a `var`/`const` block or a constructor ahead of the type they belong
to. Reordering by hand is error-prone because doc comments have to travel with
their declaration - use go/parser to get each decl's start offset, extending it
back over `GenDecl.Doc`/`FuncDecl.Doc`, and rewrite the file from those spans.
