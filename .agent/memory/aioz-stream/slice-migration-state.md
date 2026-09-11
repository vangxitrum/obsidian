---
type: project
tags: [aioz-stream, refactor, vertical-slices, aioz-template]
created: 2026-09-07
agent: main
---

Migration of `aioz-stream` from controllers/routes/services layers to vertical
slices under `internal/app/<slice>/`, on branch
`refactor/sync-with-template-source`. Scope chosen by the user: **all 14 route
domains**. Follows [[template-alignment]], which landed the contracts first.

**Done (17):** trace, report, watermark, editor, apikey, summary, player,
usage, webhook, user (owns /auth and /user), highlight, statistic, playlist,
live, video (media/captions/chapters, ~9k LOC), mail (no HTTP surface).
**The migration is complete:** `internal/controllers`, `internal/routes` and
`pkg/v1/services` no longer exist. See [[models-horizontal-split]] for the
layer split that finished it.

Spec baseline: 40 of 123 operations non-conforming, down from 100 of 100.

**The live slice hit an import cycle** and it is the one to expect again:
`live` held the controller, and the controller referenced the slice's service
types, so `live -> controllers -> live`. The fix is to move the controller into
the slice too. Its `controllers.CallerAuth` / `AllowedValues` calls become
slice-local copies.

`live` still needs the media service, which has not migrated - so it declares a
narrow `Media` interface locally (six methods) and the composition root passes
the concrete service in. Same shape as `report.MediaLookup`.

See [[repo-layout]] for the root tidy-up done alongside this.

**`internal/utils/staticlink`** signs the URLs the highlight artifacts are
served under - HMAC over the path plus an expiry, verified by a middleware on
the static group. The signing key defaults to `AccessTokenPrivateKey`, which
every replica already shares, so signing is on without another env var;
`HIGHLIGHT_STATIC_SECRET` overrides it and an empty key means unsigned. A link
issued for a directory authorizes the files under it, which is what the
thumbnail template needs.

**`internal/utils/consumer`** is the shared queue-worker chore: one pass
consumes until the broker closes the channel, then returns so `cycle` waits and
reconnects. Both webhook and the three highlight workers use it. A nil queue
yields a worker that returns immediately, so a deployment without a broker
starts instead of refusing to serve.

**Moving a file breaks path-scoped lint exclusions.** `.golangci.yml` scopes
the md5 and 0755-directory exclusions by path; migrating
`highlight_upload.service.go` into `internal/app/highlight/` made four gosec
findings reappear. Update the path patterns in the same change as the move.

**Converting a large service's errors:** `user.Service` had 71
`response.New*Error` sites. Script the mechanical ones - derive the slug from
the enclosing function name, kebab-cased, plus `-failed` / `-not-found` - and
hand-convert only the client-facing ones, which is where the reason actually
helps and where the wrong status usually hides. The apierr policy test
validates every generated slug, so a bad derivation fails the build.

`internal/routes/usage.route.go` was split: the `/payment` half became the
usage slice, and the rest was renamed `internal/routes/statistic.route.go`
because that is all it ever carried.

A slice whose service is also used by `internal/cron` (apikey, player) keeps
that method on the slice service; cron takes `*apikey.Service` /
`*player.Service` instead of the old `*services.X`.

**Things learned while migrating**
- A slice is a feature, not a URL prefix: `summary` owns
  `/media/:id/summaries*` while `media` keeps the rest of `/media`. Echo routes
  both groups fine because no two literal segments collide.
- A background loop becomes a `Chore` on `cycle.Cycle`, collected into
  `chores` in `cmd/http/main.go` and run under one errgroup, so shutdown waits
  for the pass in flight. The bare `go service.Run(ctx)` it replaced had no
  recover, so a panic inside it killed the process.
- Test a chore with `Loop.SetDelayStart()` then `Loop.TriggerWait()` - without
  the delay the startup pass runs too and every count is doubled.
- `ApiKeysSortByMap` and `PlaylistItemSortByMap` both existed and were used
  nowhere, while the wrong map was used in their place. Check for an unused
  purpose-built allowlist before writing validation for a new slice.

**Scaffolding in place**
- `internal/app/server/` - `Config`, `Module` (Name + Routes), optional `Ready`,
  `New` binds the listener eagerly, `Run` drains on ctx cancel. `/healthz` and
  `/readyz` live here because they answer for the process, not a slice.
- `internal/app/peer.go` - `Peer`, `Chore` (Name/Run/Close), errgroup over the
  API and every chore, `context.Canceled` -> nil so SIGTERM exits zero.
- `internal/utils/cycle/` ported from the template, for chores.
- `cmd/http/main.go` registers every slice on
  `slices := server.Group("", middlewares.AddLogContext())`. `apiRg` is gone -
  when media was the last user of it, `AddLogContext` (which sets the trace id)
  would have been lost entirely, so it moved onto the slices group.

**Two deviations from the template, both forced and deliberate**
1. **Repositories stay shared** in `pkg/v1/repositories` with interfaces in
   `internal/models`. The template gives each slice a `store.go`, but this is
   one relational schema - `MediaRepository` is used by 12 service files,
   `UsageRepository` by 10. Per-slice stores would duplicate them or make
   slices import each other.
2. **A slice never imports a sibling.** Where one needs another's data it
   declares a narrow consumer interface locally (see `report.MediaLookup`) and
   the composition root passes the concrete service in.

**Per-slice file set:** `common.go` (`var Error = errs.Tag("<slice>")`),
`service.go` (domain logic + Config + Validate), `request.go` (DTOs with
Normalize/Validate), `response.go` (DTOs + `type ErrorResponse =
response.GeneralResponse`), `endpoint.go` (Name/Routes/handlers + swagger),
`endpoint_test.go`.

**Gotchas**
- swag resolves a type through the **file's own imports**, so a slice that does
  not otherwise import the envelope cannot write `@Failure ... models.ResponseError`.
  Hence the `ErrorResponse` alias in each slice's response.go. `make swagger`
  exits non-zero on this - never redirect its output to /dev/null.
- `internal/controllers/dto` was folded into the editor and highlight slices;
  editor imports highlight for the media response it embeds.
- Editing `cmd/http/init.go` with regexes is dangerous - a single-line pattern
  against a multi-line call silently ate the wrong lines. Check `go build`
  after every edit to it.
- gofmt realigns the var block, so exact-string replacements against it break
  after the first edit; match with whitespace-tolerant regexes.

**Verification gate after every domain** (all four must pass):
`go build -tags purego ./...`, `golangci-lint run ./...` (0 issues),
`go test -race -tags purego ./...`, `make swagger && make swagger-verify`, then
regenerate `docs/spec_baseline.json` - it only ever shrinks (100/100 -> 87/101).

The 45+ files are **deliberately uncommitted**; the user asked to leave them.


**`internal/cron` is chores now (2026-09-07).** **One chore per job, not per
schedule**: the ten robfig entries bundled up to seven jobs each, so a slow
sweep delayed six unrelated ones and a log line could not say which was slow.
There are now 24 chores, each carrying the cron spec it runs on, collected into
`chores` in `cmd/http/main.go` and run under the same errgroup as the API.

The schedule stays a **cron spec** rather than an interval, so `@every 10s` and
`0 7 1 * *` are the same kind of thing and `cycle.Cycle` is not involved. An
interval anchored to process start drifts with every deploy - the monthly
receipt would go out on a different day each restart.

Three real defects went with it, none of them stylistic:
- the entries ran on `context.Background()`, so shutdown could not reach them
  and a deploy killed whatever was mid-pass;
- `cron.New()` was built with **no chain**, and robfig **v3 - unlike v1 - does
  not recover by default**, so a panic in any job killed the process;
- with no chain there is also no `SkipIfStillRunning`, so a pass slower than
  its interval overlapped itself and ran concurrently over the same rows.

A `cycle` runs a pass and only then waits, so overlap is impossible by
construction; `runJob` recovers per job, so one bad job does not stop the
others in its group.

**`@every` has one-second resolution** - robfig rounds anything shorter up to a
second - so a sub-second schedule is not expressible and a test using
`@every 10ms` silently never fires. Anything needing to react faster than a
second is a consumer, not a chore.

A cron schedule does not fire at registration, so the two passes that ran
eagerly before `cron.Start()` are now `Cron.Warmup(ctx)`, called once by the
composition root.

**Jobs that shared a spec now run concurrently rather than in sequence.** Within
one old entry the jobs ran in order; they no longer do. This is safe here
because the old grouping gave no real ordering guarantee anyway - with no
`SkipIfStillRunning`, consecutive passes of the same entry already overlapped -
but it is the one behavioural change to keep in mind when adding a job that
depends on another having just run. Give it its own spec offset, or fold it
into the job it follows.


**A chore belongs to the slice that owns the work.** `internal/cron` held a
wrapper per job that logged a timing line and called straight back into a slice
service, so the schedule sat one package away from the only code that could
explain it. The generic scheduled-chore type moved to
**`internal/utils/schedule`** (`schedule.NewChore(log, name, spec, fn)`) so a
slice can own its chore without importing `internal/cron`.

First one moved: `usage.NewCreateUserUsageChore`, with its own
`CreateUserUsageChoreConfig` (Enabled + Spec + Validate) and a `UsageCreator`
interface so a test drives it without a database. `internal/cron` is down to 23
jobs; the pattern for the rest is the same.

`schedule.Chore` grew `Disabled()`, mirroring `cycle.Disabled()`: an empty spec
registers no schedule and `Run` returns at once, so a chore turned off by
configuration is still constructed and collected and the shutdown sequence does
not special-case it.

**`Trigger(ctx)` is how a chore does its boot pass.** A cron schedule does not
fire at registration, and `create-users-usage` opens rows keyed by the
truncated hour, so waiting for the top of the next hour would leave the current
one unopened. The composition root calls `Trigger` once, which is also what
tests use instead of sleeping for a schedule to come round.


**`internal/cron` is gone (2026-09-08).** Every job is now a chore owned by the
slice whose service does the work, wired in `cmd/http/main.go` and driven by
`cycle.Cycle` - the template's shape, from
`~/work/templates/aioz-template`.

**`internal/utils/lifecycle`** was ported from the template. Every long-lived
component - the API and all 24 chores - registers as a
`lifecycle.Item{Name, Run, Close}`. Items start together and **close in reverse
registration order**, so chores go in first and are closed last, after the API
stops taking requests. Three things come with it, each only mattering when
something is wrong: components have names in failure logs; every goroutine
carries a **pprof label**, so a dump reads `{"component":"usage:handle"}`
instead of showing an anonymous closure; and a component that hangs on the way
down is named after `ShutdownTimeout` (15s) with a goroutine dump, rather than
hanging the process silently. It does not force anything down - killing a
component mid-write is worse than waiting.

The template's `Peer` was **dead code here** - nothing constructed it, because
`cmd/http` still uses the legacy `init()` + package globals. It now builds on
`lifecycle.Group`, and `main.go` uses a group directly. Full `Peer` adoption
still needs init.go's ~900 lines of global wiring broken up.

**`internal/utils/chore`** holds the shared shape (`Config{Enabled, Interval}`,
`Every`, `New`, log-a-failed-pass-and-continue). The template inlines this per
slice, which is right for one chore; with 24, twenty-four copies of the same
Config and the same log-and-continue rule would be twenty-four chances to get
one subtly wrong.

**`internal/utils/schedule`** remains for the two genuinely wall-clock jobs
(`usage:create-user-usage` hourly on the hour, `usage:monthly-receipt` at 07:00
on the 1st). An interval anchored to process start drifts with every deploy.
Both register as lifecycle items like everything else.

**Two defects found while moving the bodies out:**
- `uploadWaitingMediaSource` guarded re-entrancy with a plain `bool` field on
  the Cron struct - a data race, since robfig could run overlapping passes.
  `cycle` never overlaps a pass with itself, so the guard is gone.
- `deleteUsers` and the two live sweeps `return`ed on the first error, so one
  bad account or row stranded every one queued after it. They now collect with
  `errs.Group` and carry on. Within a single account the order is still
  sequential and a failure stops that account - the status is only set to
  deleted once everything behind it is gone.
- Both live sweeps logged the lookup error and then fell through to
  `len(...) == 0 { return }`, so a failing query was indistinguishable from an
  empty result. The error is returned now.

**One behaviour change:** `cycle` runs a pass immediately on start unless
`SetDelayStart`, where robfig fired nothing at registration. Every interval
chore now does one pass at boot. That subsumes the old hand-written warmup for
`handleUsage`, and these are all idempotent sweeps.


**The payment deposit watcher is a chore too (2026-09-09), and it was the last
one.** `paymentClient.StartWatchingPayment(ctx)` / `StopWatching()` are gone;
`payment.NewWatchDepositsChore` runs `PaymentClient.WatchPass` every 5s. No
periodic work in `cmd/http/main.go` is started with a bare `go` any more - all
25 chores run under the lifecycle group.

That loop had three defects, all fixed by construction:
- it selected on its **own quit channel and never on ctx.Done()**, so shutdown
  could not reach it;
- the interval was `defer time.Sleep(5 * time.Second)` - **not interruptible**,
  so stopping waited out the full five seconds;
- nothing recovered a panic.

Failures inside the pass are still logged rather than returned: it talks to a
payment gateway over the network, and one unreachable call is not a reason to
stop watching.

**Extracting a loop body: count braces, do not match `}()`.** The watcher had
three nested closures (`go func(){ ... func(){ defer sleep; ... func(){...}() }() }()`),
and taking the last `}()` produced a syntactically broken file. Find the
opening brace and walk the text counting depth.

Also removed once nothing referenced them: `MediaService.StopCron` and the
three `stopWatch*Ch` channels it closed - the loops that selected on them are
chores now.


**Controller naming is gone entirely (2026-09-09), files included.** The types
`MediaController`, `MediaCaptionController`, `MediaChapterController` and
`LiveStreamController` became `*Endpoint`, and the files followed:

    media/handlers.go          -> media/media_endpoint.go
    media/caption_handlers.go  -> media/caption_endpoint.go
    media/chapter_handlers.go  -> media/chapter_endpoint.go
    live/handlers.go           -> live/stream_endpoint.go

`endpoint.go` stays the slice's route registrar - the `Endpoint` type with
`Name()`/`Routes()` that every slice has - and the handler holders sit beside it
in `<thing>_endpoint.go`. The composition root therefore names the registrar
`mediaEndpoint` and the handler holders `mediaHandler` /
`mediaCaptionHandler` / `mediaChapterHandler`: renaming both to `mediaEndpoint`
collided, which is the one thing to watch for when doing this rename.

Use `git mv` so the history follows: git recorded all four as `R` renames
rather than delete+add.
