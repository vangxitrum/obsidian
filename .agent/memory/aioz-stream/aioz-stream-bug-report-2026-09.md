---
type: fact
tags: [security, bugs, aioz-stream, sqli, auth]
created: 2026-09-07
agent: main
---

Defects found reading `aioz-stream` at `10b1f8ea` (2026-09-07). All were fixed on
`refactor/sync-with-template-source`; see [[template-alignment]] for the wider
change. Recorded here because several are non-obvious and could be reintroduced.

**Security, reachable over HTTP**
- The `stream-trace-id` request header was used verbatim as a filename
  (`middlewares/log.go` -> `response.TraceHelper.Save`), giving an arbitrary
  `.json` file write outside `./trace_data` to anyone who could make a request
  fail. Fixed with `^[A-Za-z0-9_-]{1,64}$` at both the middleware and the helper.
- `GET /api/trace/:id` had **no middleware at all** and served the captured
  request - headers included. `ExcludedHeaders` dropped only `Authorization`, so
  `stream-secret-key`, `Admin-Api-Key` and `Cookie` were written to disk and
  served back anonymously. Now behind a new `middlewares.RequireAdminApiKey`
  (constant-time, fails closed when unconfigured) and the header deny-list covers
  every credential header.
- SQL injection via `order_by` on `POST /api/analytics/metrics/bucket/...`: the
  `OrderMap` check was nested inside `if payload.SortBy != ""`, so omitting
  `sort_by` skipped it and the value reached a raw GORM `Order()`. GORM's
  `Order(string)` emits `clause.Column{Raw: true}` - verbatim SQL, no escaping.
- SQL injection via `filter_by.countries`: `models.MetricFilter.BuildQuery()`
  built `'...'` literals with `strings.Join` and no escaping, and `Countries` was
  validated **nowhere in the repo**. `device_types`/`tags` were checked only on
  the aggregated endpoint. Fixed with `quoteLiteral` plus one shared
  `validateMetricFilter` used by all three analytics handlers.
- `Authorization: Bearer` with no token indexed `fields[1]` after checking only
  `len(fields) != 0` - panic on two paths in `AuthenticateAPIKey`.

**The one most likely to be user-visible**
An API key with `ttl=0` means "never expires", and `NewApiKey` stamps
`ExpiredAt == CreatedAt` for it. The `stream-public-key`/`stream-secret-key`
path checked only `time.Now().After(ExpiredAt)`, so **every ttl=0 key was
rejected as expired** through the headers while working fine through Basic auth.
The two branches were ~70 duplicated lines; they are now one
`authenticateApiKey` helper and the `numTtl > 0` guard is the single rule.

**Quietly dead code**
- `NewTraceHelper` ran `MkdirAll` then returned the original not-exist error
  anyway, so the trace subsystem never initialized on a host without
  `./trace_data`.
- `ExtractRequestData` had `if body == nil { io.LimitReader(body, ...) }` - the
  condition inverted, so bodies were never captured and would have nil-panicked
  if the branch ever fired.
- `TraceHelper.Total` was never incremented by `Save`, so `Clean` never pruned;
  a 1-second ticker did a full `ReadDir` forever.
- `TestSeed` (in `seeds/`) writes 500k rows into the database from `../debug.env`
  under a plain `go test ./...`. Now gated on `SEED_DATA=1`.
- Three `internal/utils/*client*` test files drive a live host with a hardcoded
  API key committed in the source. Gated on `LIVE_CLIENT_TESTS=1`; **the
  credentials are still in git history**.
- `internal/utils/errors/` declared `package middlewares`. Deleted, unused.
- `TestManifestGen_DoesNotMarkMediaProcessedWhenGenerationFails` panicked: a
  later commit added a nil-queue guard that short-circuits before the path the
  test asserts. Its `fakeFsStorage` also raced under `-race` (unsynchronised
  slice appends, four concurrent callers).

**Config**
`MustNewAppConfig` used viper with no `AutomaticEnv`/`BindEnv`, so real
environment variables were ignored entirely - only `./$APP_ENV.env` was read.
Both panics discarded the underlying error. The 21 `required:"true"` /
`validate:"required"` tags were decorative; a `validateRequired` reflection check
now runs after unmarshal. **Deploy risk:** a deployment whose env file is missing
one of those keys will now fail at boot instead of failing later.

---

## 2026-09-07, second pass: the same defects swept across the whole codebase

The four bugs found while writing the first slices turned out to be classes,
not incidents. Each is now fixed everywhere and pinned by a test.

**Unchecked `ctx.Get("authInfo").(models.AuthenticationInfo)` - 74 sites** in 12
controllers. A handler reached without the auth middleware panicked; the
recover middleware turned that into a 500 with a stack trace instead of a 401.
Five more sites checked `!ok` but not `authInfo.User == nil`, which is the same
panic one dereference later, and one of those answered **400** for an
authentication failure. All now go through `controllers.CallerAuth` /
`CallerId`. `internal/controllers/auth_test.go` walks the module with go/parser
and fails on any one-value assertion of authInfo - verified to catch a planted
offender.

**Sort allowlists naming columns that do not exist.** These pass validation and
then reach a raw `ORDER BY`, so the endpoint answers 500 on a value it just
told the client was valid. `internal/models/sortby_test.go` now derives each
model's real columns by reflection over its gorm tags and checks every map:

- `PlaylistSortByMap` allowed `title` and `status`; `Playlist` has neither, and
  the one real column, `name`, was being **rejected**. Worse, the map was
  shared by two handlers with different tables - and
  `PlaylistItemSortByMap`, which matches the items query exactly, was **used
  nowhere**. `GetPlaylistById` now uses the item map, `GetUserPlaylists` the
  corrected `{created_at, name}`.
- `GET /live_streams/{id}/streamings` validated against the generic
  `SortByMap` (`name`), but `live_stream_media` has `title`. Now uses
  `LiveStreamMediasSortByMap`.
- The generic map was also applied to `watermarks` (column is
  `watermark_name`) and `content_reports` (no name column at all). Both now
  have their own map. **These two were carried forward into the new slices, so
  the guardrail caught my own migration, not just the legacy code.**

**Error messages drifting from the maps they describe.** Hardcoded "Allowed
values: ..." strings had gone stale - playlist advertised `name` while
rejecting it. `controllers.AllowedValues(map)` renders the text from the map,
so it cannot drift; all six sites use it.

**Short reads before a content sniff.** `src.Read(buf)` is not guaranteed to
fill, and the unfilled tail was sniffed as trailing zeros. Both sites
(`validate.IsValidateThumbnail`, the player-theme logo upload) now use
`io.ReadFull`, tolerate the two EOF cases, sniff only the bytes actually read,
and answer 400 rather than 500 for an empty or undecodable file.

**Admin authorization was a compiled-in list of five email addresses**
(`models.ValidEmail`, two of them personal Gmail accounts) with **no config
path** - `ADMIN_MAIL_LIST` only decides where notification mail is sent. Now
`ADMIN_EMAILS` in config, via `validate.SetAdminEmails`; the compiled-in list
remains the default so an unconfigured deployment is not locked out, and boot
logs a warning naming the addresses. The comparison is now case-insensitive and
space-trimmed - it was an exact map lookup, so `Tue.Phan@aioz.io` was refused
while the lowercase spelling worked.

---

## Found while migrating apikey and summary (2026-09-07)

**A background goroutine could crash the whole process.**
`MediaSummaryService.Run` read `existed.Language` on a path where `existed` is
nil - `GetSummaryByTaskId` returning `ErrRecordNotFound` leaves it nil, the
`if existed != nil` block is skipped, and the translate loop below dereferences
it anyway. It ran as a bare `go mediaSummaryService.Run(ctx)`; echo's recover
middleware covers HTTP handlers only, so the panic was fatal. Now a
`summary.Chore` that logs a failed pass and takes the next interval.

**`DELETE /api_keys/{id}` on another tenant's key answered 200 having done
nothing.** The existence check was unscoped, the delete scoped, and GORM
reports no error when an update matches no rows. The unscoped lookup also told
a caller whether an arbitrary key id exists. Both paths now go through
`owned()`, which answers 404 for a key the caller does not own - not 403,
which would confirm it exists.

**`GET /media/{id}/summaries`: `if Offset < 0 && Limit < 0`** - should be `||`.
A negative offset alone passed, and unlike the limit it was never clamped, so
it reached the query as `OFFSET -1` and answered 500.

**`DELETE /media/{id}/summaries/{lan}` validated a truncated language.** A
regexp took the first two characters, checked those, then passed the whole
parameter to the store - so "envil" validated as "en" and deleted nothing.

**Four `@name` annotations were copy-pasted between types.**
`UpdateMediaSummaryData` was named `CreateMediaSummaryData`,
`UpdateMediaSummaryResponse` named `CreateMediaSummaryResponse`,
`RequestUpdateSummary` named `RequestCreateSummary`, and
`GetMediaSummarysRequest` named `GetMediaSubtitlesData`. The update models
collided with the create ones, so the generated SDK had no update types at all.

**`MediaSummaryService.DeleteMediaSummary` returned the store's error
unwrapped**, so a missing summary and a broken database both answered 500 with
no reason attached.

---

## Found while migrating player themes (2026-09-07)

**Replacing a player-theme logo could destroy it.** `UploadPlayerThemeLogo`
deleted the existing object from storage *before* uploading the replacement, so
an upload that failed afterwards left the theme row pointing at an object that
no longer existed - a broken logo with nothing to restore it from. The slice
uploads, records and repoints first, and only then removes the old object; the
regression test asserts the call order and that a failed upload leaves the
existing logo untouched.

**Account teardown wedged on accounts that used the feature.**
`DeleteUserPlayerThemes` (called from the delete-users cron) looped over a
user's themes calling the single-theme delete, which refuses a theme still
applied to media. So deleting an account that had ever attached a theme failed
every night. `DeleteAllForUser` now skips the in-use check, since the media
using those themes is being deleted in the same sweep.

**The logo response set `Cahche-Control`** - misspelled, so no cache directive
ever reached a client and every player refetched the logo.

**`GetPlayerThemeById` and `ListAllPlayersThemes` built the logo URL two
different ways** - one with an inline `fmt.Sprintf("%s/api/players/%s", ...)`,
the other with `models.PlayerLogoUlrFormat`. They agree today; nothing made
them.

---

## Found while migrating usage and webhook (2026-09-07)

**The webhook consumer spun at 100% CPU after any broker disconnect.**
`startCallWebhookWorker` ran `for { for mess := range messCh {} }`. Ranging a
closed channel returns immediately, so once RabbitMQ closed the channel the
outer loop never blocked again. It also started from the service
**constructor**, on `context.Background()` - so shutdown could never reach it -
and `panic(err)` if the broker was unreachable, taking the process down at
boot. Now `webhook.Consumer`, a chore on `cycle.Cycle`: one pass consumes until
the channel closes, then returns and the cycle waits before reconnecting.

**A subscriber answering 5xx counted as a successful delivery.**
`handleWebhook` only treated a transport error as failure; any HTTP status was
accepted, so a subscriber returning 500 was never retried and the failure was
invisible. A non-2xx is now a failed delivery.

**PATCHing `encoding_failed` or `partial_finished` silently did nothing.**
`WebhookService.UpdateWebhook` rebuilt `models.UpdateWebhookInput` and left
those two flags out, so the store never saw them. The client got a 200.

**Webhook update and delete could report success for a webhook the caller does
not own** - the same unscoped-check / scoped-write shape as the api_key delete.
Both now read first.

**Webhook retries could never be exhausted.** `HandleWebhookRetry` looked the
delay up in a map keyed 1..5 and, when the count went past 5, left the row with
its old `NextRetryAt` - so it was retried on every sweep forever. The slice
drops the row when the attempts run out.

**`/payment/top_ups` and `/payment/billings` never clamped `limit`.** It was
defaulted when zero but otherwise passed through to a query that interpolates
`LIMIT %d`.

**`UsageService.GetUserTopUps` returned the store's error unclassified**, so a
database failure reached the client as a bare 500 - unlike the billings
endpoint beside it, which classified the same failure.

**`usage.service.go` and `live_stream.service.go` import `zerolog`** in a
codebase that otherwise logs with `log/slog`; usage had exactly one zerolog
call, now slog. live_stream still has five (it has not been migrated yet).

---

## Found while migrating auth and the account (2026-09-07)

**Deleting an account accepted an API key.** `/user` DELETE sat behind the
session middleware, but the surrounding group made it easy to reach with the
API-key one; the slice now puts it behind the session middleware explicitly and
a test pins that an API key is refused. An API key is a machine credential
kept in configuration - it should not be able to destroy the account it
belongs to.

**Three statuses disagreed about the same thing.** "User already existed"
answered **404**; "User not found" answered **400** in two places and 404 in a
third. Now `account-already-exists` is a 409 and `account-not-found` a 404
everywhere.

**`NewUserService` wrote to the database from the constructor** -
`GenerateExclusiveCodes(context.Background())`, logging on failure - so a
seeding failure was a log line nobody owned, on a context nothing could cancel.
Now an explicit `Prepare(ctx)` the composition root calls.

**`/user/subscribe` answered success for a malformed address.**
`mails.FormatEmail` failed, the error was discarded, and the handler returned
"Subscribe successfully." regardless - so a typo looked like a subscription.
`CreateSubscribeInfo` also returned nothing at all, so no caller could tell a
duplicate from a broken database. It now returns an error, and a duplicate is
still success.

**The exclusive-program form answered one message for every omission.** Four
required fields, one "X is required" per branch but no indication which; the
slice names the field in the message and in a `field` log attribute.

**`EmailSignUpData` / `EmailSignUpResponse` were declared and never used** -
the sign-up handler returned the login DTO. Identical shape, so no wire
difference, but the spec described a type nothing produced.

---

## Found while migrating highlight media (2026-09-07)

**A broker reconnect silently stopped every highlight upload.** The three
consumers (`consumeAssembling`, `consumeManifestGen`,
`consumeTaskGetSemanticTimeLines`) each returned when their delivery channel
closed and were never started again - the mirror image of the webhook spin.
Uploads simply stopped being assembled, with no error anywhere and nothing to
notice. All three are chores on the shared `internal/utils/consumer` now.

**A missing broker stopped the API serving at all.** `StartConsumers` returned
`ErrInvalidHighlightQueue` when any queue was nil, and `cmd/http/main.go`
treated that as fatal - it cancelled the run context and skipped starting the
playlist watcher, the summary sweep and the HLS import. A broker that was not
configured therefore took down unrelated features. The workers now stop
individually and the rest of the service runs.

**The same sentinel got two different statuses.** `ErrInvalidMediaStatus`
answered 409 from `CompleteUpload` and fell through to 500 from
`UploadHighlightMediaChunk`. Several branches also returned a 500 whose
client-facing message was the raw sentinel text. One `classify` function now
maps every sentinel once.

**Validation branches passed an already-handled `err` as the internal error.**
Five sites in the create and chunk handlers did `NewHttpError(400, err, "...")`
where `err` was the nil left over from a previous `uuid.Parse` - harmless, but
it means none of those 400s carried a cause.

**Note, not fixed:** the highlight static mount
(`/api/highlight-media/static`) serves the storage root with no authentication.
There is no directory listing, so artifacts are reachable by unguessable id
only - the same posture as a CDN. Left as is because the editor depends on it.

---

## 2026-09-07, static artifacts and analytics

**Highlight artifacts were served to anyone holding the URL, forever.** The
static mount had no authentication - it cannot use the API's, because the
editor loads these into `<video>` and `<img>` elements, which send no headers -
so the only protection was that paths contain a uuid. A link in a proxy log, a
referrer header or a screenshot was a permanent grant.

Fixed with signed links: `internal/utils/staticlink` HMACs the path plus an
expiry, a middleware on the static group verifies it, and `GetStaticLink` signs
on the way out. The client is unaffected, because the server produces every
link. The key defaults to `AccessTokenPrivateKey` - already required, already
secret, already identical across replicas - so signing is on by default rather
than waiting for one more environment variable.

**Operator metrics were reachable by anyone who knew the path.**
`/api/metrics/summary` and `/api/metrics/media` had no middleware; the admin
key was checked inside the service, so a route added to that group later would
have had no check at all. The gate is on the group now, and the service's
refusal is a 403 rather than falling through to a 500.

**`GetStatisticMedias` clamped a negative limit to 50 rather than rejecting
it** - `if payload.Limit < 0 { payload.Limit = 50 }` - while a zero limit fell
straight through to the query. Both are normalized now.

---

## Found while migrating playlists (2026-09-07)

**Ownership refusals leaked existence.** Five sites answered **403 "You are not
allowed to ..."** for a playlist the caller does not own, which confirms that
the playlist exists. They answer 404 now, the same as a playlist that is not
there - the same change made to the api-key slice.

**A reorder that would corrupt the list answered 400.** `MoveItemInPlaylist`
validates the resulting linked list and refuses when it would not be a list;
that is a state conflict (409), not a malformed request, and the three
neighbour-adjacency refusals beside it are the same.

**A playlist that has reached its item limit answered 400**, and media still
transcoding answered 400. Both are 409: the request fights the current state
and the caller can act on it.

**`AddMediaToPlaylist` did not de-duplicate its inputs.** The path names one
playlist and the body may name more; naming the same playlist or media twice
counted it twice against the item limit.

---

## Found while migrating live streaming (2026-09-07)

**The live-stream webhooks were never authenticated, although the token
existed.** `STREAM_WEBHOOK_TOKEN` is in `internal/config` and in
`env-example/app.env`, and the mediamtx config sends `webhook_token=...` on
both `runOnConnect` and `runOnDisconnect` - but **no Go code ever read it**.
The two endpoints are public and end broadcasts, so anyone who could reach the
API and guess a connection id could hang up someone else's stream.
`requireWebhookToken` now checks it in constant time. An unset token keeps the
old behavior, with a boot warning, so this is a configuration change rather
than a breaking one.

**A third 100% CPU spin, in the end-of-stream watcher.**
`StreamService.startEndLiveStream` had the same `for { for mess := range
messCh {} }` as the webhook worker, plus `panic(err)` on a consume failure, and
it was started from the constructor on `context.Background()`. Beyond the
spin, it meant a broker reconnect stopped streams being finished at all - they
stayed marked live forever. Now a chore on the shared consumer.

**A "cannot delete while streaming" guard that never fired.**
`DeleteLiveStreamMulticast` nested the check inside the error branch:

    if err != nil {
        if !errors.Is(err, gorm.ErrRecordNotFound) { return ... }
        if liveStreaming != nil { return "can't delete, it is streaming" }
    }

`liveStreaming` is only non-nil when `err == nil`, so the guard ran only when
the lookup had failed. A multicast that was actively streaming could always be
deleted.

**Ownership refusals answered 400 or 401 across nine sites** ("You are not
allowed to ..."), which both misstates the failure and confirms the resource
exists. They answer 404 now, matching the rest of the API.

**`zerolog` is gone from the codebase.** The last two importers were
`live_stream.service.go` and `usage.service.go`; both now use `log/slog`.
