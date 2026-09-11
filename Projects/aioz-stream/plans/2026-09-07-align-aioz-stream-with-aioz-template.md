---
title: Align aioz-stream with aioz-template
date: 2026-09-07
project: aioz-stream
status: implemented
tags: [plan, aioz-stream, aioz-template, security]
---

# Align aioz-stream with aioz-template (contracts first) + bug remediation

## Context

`~/work/templates/aioz-template` is the house Go service scaffold. Its README is the
spec, and nearly every rule in it is backed by a test that fails when the rule is
broken. `aioz-stream` predates it: ~200 Go files, viper env config, package-global
wiring inside a 795-line `func init()`, Echo controllers/services/repositories, and no
error-classification layer at all.

A full restructure to the template layout (cobra + aioz-config/logger/stats, vertical
slices, `Peer` lifecycle) would touch every file of a live service. Scope chosen with
the user: **adopt the template's enforceable contracts without moving files.** The
existing `internal/controllers/`, `pkg/v1/services/`, `pkg/v1/repositories/` layout
stays. What changes is the error contract, the HTTP error envelope, the guardrail
tests, and the build/CI surface.

Alongside that, the exploration turned up real defects - including two that are
security-relevant and reachable over HTTP. The user asked for these to be fixed here,
each with a reproducing test written first.

Branch `refactor/sync-with-template-source` currently has **no** work on it: it is
byte-identical to `feat/migrate-with-m3u8` (`10b1f8ea`). `go build -tags purego ./...`,
`go vet -tags purego ./...` and `go test ./...` all pass at HEAD.

---

## Part 1 - Bug report

Every item below was read in the tree at `10b1f8ea`. Severity is about reachability
from an HTTP request, not about how hard the fix is.

### Critical - reachable from a request

**B1. `stream-trace-id` header is used verbatim as a filename → arbitrary file write.**
`internal/middlewares/log.go:21` takes `traceId` straight from the client-supplied
`stream-trace-id` header (only generating a UUID when the header is absent).
`internal/utils/response/response.go:97-101` passes it to
`TraceHelper.Save`, which builds `fmt.Sprintf("%s/%s.json", t.Output, traceId)`
(`internal/utils/response/trace.go:84`) with no validation. Any request that produces
an error, carrying `stream-trace-id: ../../../whatever`, writes a JSON file outside
`./trace_data`. Reachable pre-auth, since `ResponseError` runs on auth failures too.

**B2. `GET /api/trace/:id` is unauthenticated and returns captured request data.**
`internal/routes/trace.route.go:21-25` registers the route with no middleware.
`internal/controllers/trace.controller.go:26` reads `ctx.Param("id")` into
`TraceHelper.Load`, same unsanitized path join (`trace.go:102`). The stored
`RequestInfo` includes every request header except `Authorization`
(`internal/utils/response/response.go:157-159`) - so `stream-secret-key`
(`internal/models/variable.go:20`), `Admin-Api-Key` and `Cookie` are all written to
disk and served back to any anonymous caller who can guess or observe a trace id. Trace
ids are client-chosen (B1), so guessing is not required.

**B3. SQL injection via `order_by` on the analytics endpoints.**
`internal/controllers/statistic.controller.go:400-408` nests the `models.OrderMap`
check **inside** `if payload.SortBy != ""`. Omit `sort_by` and `order_by` is never
validated; it reaches
`pkg/v1/repositories/statistic.repository.go:641`, `:676`, `:906` as
`Order(fmt.Sprintf("%s %s", input.SortBy, input.OrderBy))`. GORM v1.25.12's
`Order(string)` emits `clause.Column{Raw: true}` - verbatim SQL, no escaping.
Route: `POST /api/analytics/metrics/bucket/:metric/:breakdown`
(`internal/routes/usage.route.go:53`).

**B4. SQL injection via `filter_by.countries` (and `device_types`, `tags`).**
`internal/models/action.go:113` `MetricFilter.BuildQuery()` builds SQL string literals
by hand: `fmt.Sprintf("'%s'", strings.Join(values, "','"))`, with no quote escaping. The
result is concatenated into a raw query handed to
`Table(fmt.Sprintf("(%s) as sub", query))` at `statistic.repository.go:287, 466, 638,
868, 1083, 1282`. `Countries` is validated nowhere in the repo. `device_types` and
`tags` are validated only on `GetAggregatedMetrics`, not on the breakdown or timeseries
handlers.

B3 and B4 both sit behind `authWithApiKey`, so they need a valid API key - they are
privilege escalation from any tenant to the whole database, not anonymous RCE.

**B5. Index out of range on a malformed `Authorization` header.**
`internal/middlewares/auth_middleware.go:143-145` and `:170-171` both index `fields[1]`
after checking only `len(fields) != 0`. A request with `Authorization: Bearer` or
`Authorization: Basic` (no value) panics. `middleware.Recover` at `cmd/http/main.go:72`
turns it into a 500 rather than a crash, but every such request logs a stack trace.
Contrast `getUserIdFromToken` at `:110`, which correctly checks `len(fields) == 2`.

### High - wrong behaviour, not exploitable

**B6. `TraceHelper` never initializes when its directory is missing.**
`internal/utils/response/trace.go:43-51`: the `return err` sits outside the
`os.IsNotExist` branch, so after `MkdirAll` succeeds the function still returns the
original not-exist error. `cmd/http/init.go:588` logs it and continues, leaving
`TraceHelperInstance` nil. The whole trace subsystem is dead on any host where
`./trace_data` does not already exist.

**B7. Request-body capture is dead code, and would panic if it ran.**
`internal/utils/response/response.go:191-198`: `if body == nil { io.LimitReader(body, 1024) }`
- the condition is inverted. `http.Server` never hands a nil `Body`, so the branch never
fires and `RequestInfo.Body` is always empty; if it ever did fire, `io.ReadAll` on a
`LimitReader` wrapping nil dereferences nil.

**B8. `TraceHelper.Total` is never updated, so cleanup never runs.**
`Save` (`trace.go:83-99`) writes a file but does not add its size to `t.Total`. `Clean`
(`:138`) breaks out unless `t.Total >= MAX_TRACE_FOLDER_SIZE` **and** `count > 5`.
`Total` is fixed at whatever the directory held at startup, so the folder grows without
bound. The 1-second ticker (`:69`) meanwhile does a full `ReadDir` every second forever.
`Clean` also ignores the `Info()` error in its sort comparator (`:129-134`) - a file
removed concurrently gives a nil `FileInfo` and panics on `.ModTime()`.

**B9. `TraceHelper.Load` can short-read.** `trace.go:116` uses a single `file.Read` into
a `stat.Size()` buffer instead of `io.ReadAll`, and ignores the returned byte count.

**B10. `ResponseError` panics when `HTTPError.Message` is nil.**
`internal/utils/response/response.go:84`: `reflect.ValueOf(httpErr.Message).Type()`
panics on a nil interface. Any `&echo.HTTPError{Code: x}` built without a message
reaches it. Same shape at `:38` in `ResponseSuccess` for a typed-nil `data`.

**B11. A 5xx leaks its internal cause to the client.**
`response.go:87-91`: when `httpErr.Message` is empty the internal error's text is copied
into the client-facing `message`. `NewHttpError` (`:51-58`) does the same fallback at
construction. The template's rule is the opposite - a 5xx returns the reason slug only,
and the cause goes to the log (`aioz-template/README.md:580-591`). Live example:
`trace.controller.go:29` returns `NewHttpError(500, err)` with no message, so the
filesystem path of the trace directory is returned to the caller.

**B12. `ResponseFailMessage` reports a 500 as `"fail"`.** `response.go:136-139` assigns
`status = "fail"` unconditionally, then re-assigns the same value inside a 4xx check.
`ResponseError` uses `"error"` for 5xx. So two paths render the same status code with
two different envelopes - and `auth_middleware.go:352-356` takes the wrong one.

**B13. Three middleware paths bypass the response envelope entirely.**
`auth_middleware.go:152`, `:264`, `:334` return an `*echo.HTTPError` as a plain `error`.
There is no custom `HTTPErrorHandler` registered anywhere (grep confirms), so Echo's
default handler renders `{"message": "..."}` instead of `GeneralResponse{status, message}`.

**B14. API-key expiry is enforced differently depending on the transport.** The Basic-auth
branch checks `time.Now().After(existed.ExpiredAt) && numTtl > 0`
(`auth_middleware.go:238`); the header branch checks only `ExpiredAt` (`:308`). Same key,
two behaviours. The two branches are ~70 lines of otherwise-duplicated code
(`:194-266` vs `:276-336`), and this is the divergence that duplication produced.

**B15. A bookkeeping write failure fails the request.** `auth_middleware.go:260-265` and
`:330-335` return 500 when `UpdateApiKeyLastRequestedAt` fails. `DeserializeUser` only
logs the same class of failure (`:76-82`).

**B16. A malformed `user_id` claim yields 500, or silently yields the zero UUID.**
`auth_middleware.go:122-125` returns `NewInternalServerError` for an unparseable claim
(should be 401). `:155` does `userId, _ = uuid.Parse(...)`, discarding the error, so a
bad claim proceeds with `uuid.Nil` and is only caught later by the user lookup.

### Medium - conventions and operations

**B17. `internal/utils/errors/errors.go` declares `package middlewares`.** Directory says
`errors`, package clause says `middlewares` (`:1`). Importable only under an alias; the
two vars in it appear unused.

**B18. `/metrics` is served on the public API port.** `cmd/http/main.go:60`, no auth. The
template puts metrics, pprof and `/version/` on a separate debug listener that is never
exposed (`aioz-template/README.md:248-258`). pprof here is already correctly bound to
loopback (`main.go:48-57`) - metrics was not given the same treatment.

**B19. `make build` does not pass `-tags purego`, but the lint config assumes it does.**
`.golangci.yml:27-31` says "`make build` compiles with -tags purego, so lint must analyse
the same build", and `.gitlab-ci.yml:132` uses `go list -tags purego`. `Makefile:63-64`
has no tag. Lint and CI analyse a build configuration the Makefile does not produce.

**B20. No version stamping.** `Makefile:63` builds with no `-ldflags`, so there is no
release/sha/build stamp and no dev-vs-release defaults channel
(`aioz-template/README.md:667-710`). Nothing in the repo can report which commit is
deployed.

**B21. `make test` has no `-race`, and CI has no test stage.** `Makefile:50-51` is
`go test ./... -v`. `.gitlab-ci.yml` stages are lint/build/deploy only - the 21 test
files never run in CI.

**B22. The committed swagger spec is never verified.** `make gen-swagger`
(`Makefile:113-128`) is interactive (`read -p`), so it cannot run in CI, and there is no
`swagger-verify` equivalent. `docs/swagger.json` can drift from the handlers silently.
`docs/liblab.config.json` is still unmodified boilerplate pointing at the petstore
example.

**B23. Config ignores real environment variables.** `internal/config/config.go:141-151`
calls `viper.SetConfigFile` + `ReadInConfig` with no `AutomaticEnv`/`BindEnv`. Setting
`PORT=...` in a container's `environment:` has no effect; only the `./$APP_ENV.env` file
is read. Both panics also discard `err`, so "can not read config file" is the entire
diagnosis.

**B24. `required:"true"` / `validate:"required"` config tags are decorative.** Nothing
runs a validator over `AppConfig`; a missing required key surfaces as a nil-pointer or a
panic deep in a helper constructor.

**B25. Config is copied into package-level mutable globals.** `config.go:153-196` writes
into `models.BeUrl`, `models.AdminMailList`, `models.DemoVideoId`,
`models.BetterStackToken`, the rate limits and the hub costs.

**B26. `SIGTERM` is not handled.** `cmd/http/main.go:122` watches `os.Interrupt` only, so
a container stop gets no graceful shutdown. Also `paymentClient.StartWatchingPayment`
(`:66`) and `cron.Start()` (`:69`) run on `context.Background()`, outside the shutdown
context, and the early `return` at `:127` skips all cleanup.

**B27. Duplicate construction in the composition root.** `cmd/http/init.go` builds
`mediaRepo`/`userRepo`/`emailConnectionRepo`/`walletConnectionRepo` twice (`:264-303`),
`summaryRepo` twice (`:361`, `:376`), `playlistController` twice (`:872`, `:888`).

**B28. `go.mod` says `go 1.22`, CI runs Go 1.25.** `go.mod:3-4` vs
`.gitlab-ci.yml` `GO_IMAGE=registry:5000/go:1.25`.

---

## Part 2 - Alignment work

### 2.1 `internal/utils/apierr` - port from the template

New package, ported from `aioz-template/internal/utils/apierr/`
(`apierr.go`, `reason.go`, `trace.go`) with the module path adjusted. Adds
`github.com/zeebo/errs/v2` to `go.mod`.

Surface to keep exactly as the template has it, because the guardrail tests and the
README recipes depend on it: `New/Wrap/WrapWith/Wrapf`, the status constructors
(`BadRequest Unauthorized Forbidden NotFound Conflict TooManyRequests Internal
Unavailable`), `With(kv ...any)`, `ClientMessage()`, and the readers
`Status/Reason/Fields/Tags/Trace`. The wrap forms return `error`, not `*Error`, so a nil
`*Error` never becomes a non-nil interface.

The one rule: **a reason names the subject, not the status** - `contract-not-found`,
never `not-found`.

### 2.2 One error handler, one envelope

New `internal/utils/response/errorhandler.go` (kept in `response/` rather than a new
`internal/app/server/`, since the transport package does not exist here), ported from
`aioz-template/internal/app/server/middleware.go:95-198`, plus `humanize` from
`message.go:23-58`.

Registered as `server.HTTPErrorHandler` in `cmd/http/init.go` next to `server = echo.New()`
(`:974`). Behaviour, from the template:

- an `apierr` renders as classified; anything unclassified is a 500
- **4xx** sends the author's message, capitalized and full-stopped by `humanize`, with
  the leading-identifier exemption
- **5xx** sends the reason slug and the request id only; the message is replaced, and the
  cause, `Tags` and `Trace` go to the log (fixes **B11**)
- every failure increments a `request_failures` metric tagged by route and reason

**Envelope compatibility is the constraint here.** The template emits
`{"error":{code,reason,message,request_id}}`; this API emits
`{"status","message","data"}` and has live SDK/frontend consumers. Keep
`GeneralResponse` as the outer shape and add the new fields inside it:

```json
{"status":"fail","message":"Name must be at most 64 bytes.","reason":"name-too-long","request_id":"..."}
```

`status` keeps its current derivation (`fail` for 4xx, `error` for 5xx) and
`ResponseFailMessage` is corrected to match (**B12**).

`internal/utils/response/errors.go` and `NewHttpError` stay as a deprecated shim
delegating to `apierr`, so the ~20 unconverted controllers keep compiling. New code uses
`apierr` directly. `internal/utils/errors/errors.go` is deleted (**B17**).

### 2.3 Guardrail tests

Ported from the template, each one guarding a mistake invisible at the call site:

| new test | ported from | catches |
| --- | --- | --- |
| `internal/utils/apierr/policy_test.go` | template `apierr/policy_test.go` | a reason slug that restates its status. Walks the module with `go/parser`; skip list must add `mediamtx/`, `w3streamcore/`, `docs/`, `internal/proto/` |
| `internal/utils/apierr/apierr_test.go` | template `apierr/apierr_test.go` | slug format rules |
| `internal/utils/response/message_test.go` | template `server/message_test.go` | a message reaching a user uncapitalized or unpunctuated |
| `internal/utils/response/diagnostics_test.go` | template `server/diagnostics_test.go` | a 5xx leaking its cause, or leaving no stack in the log |
| `docs/spec_test.go` | template `docs/spec_test.go` | an operation with no `operationId`, no `@Tags`, or no documented 500 |

`docs/spec_test.go` will fail loudly on first run against the existing spec. Land it
skipped against a checked-in baseline of currently-failing operation ids, so it blocks
new violations without requiring every existing handler to be annotated in this change.

### 2.4 Build and CI parity

`Makefile`:

- `build`: add `-tags purego` (**B19**), `-trimpath`, `CGO_ENABLED=0`, and
  `-ldflags` stamping `release`/`sha`/`build`/`channel` into a new
  `internal/utils/version` package ported from the template (**B20**). Those four must
  stay **plain `string` vars** - `-X` aimed at a struct field is silently dropped.
- `test`: `go test -race -tags purego ./...` (**B21**).
- new `swagger`: non-interactive `swag init --parseDependency -g cmd/http/main.go -o docs -q`
  then `swag fmt`, with the pin `SWAG_VERSION`. Keep `gen-swagger` as an alias for a
  transition period, or drop it - it cannot run unattended (**B22**).
- new `swagger-verify`: regenerate into a temp dir and diff `docs/swagger.json`.
- `.PHONY` covering all of the above.

`.gitlab-ci.yml`: add a `test` stage between `lint` and `build`, running
`make swagger-verify` then `make test`, with `CGO_ENABLED=1` (race needs cgo) and the
existing `.go-private` anchor. Add `./bin/api version` as a smoke test in `build-job`,
which is what catches ldflags that silently did not land.

`internal/utils/version/version.go` + `version_test.go` ported from the template; a
`/version` route on the API for now (a separate debug port is out of scope for this pass,
so **B18** is addressed by moving `/metrics` behind an auth check rather than by adding a
second listener - flag this in review as a partial fix).

### 2.5 Slice conventions applied where they are cheap

Not restructuring, but adopt per-package where a file is touched anyway:

- one `var Error = errs.Tag("pkg")` per package, `Error.Wrap` at boundaries
- `*slog.Logger` as the first constructor argument, `log.With("component", ...)` for
  sub-loggers, no new package-level logger globals
- request DTOs and response DTOs in separate types, each with a trailing `// @name X` -
  `internal/controllers/dto/` already does this for the highlight/editor features and is
  the model to follow

---

## Part 3 - Bug fixes

Per the repo rule, **each fix starts with a failing end-to-end test**, driven through the
real Echo handler with `httptest` rather than by calling the function directly. The
existing `internal/routes/highlight_media_test.go` and
`internal/utils/hlsimport/import_e2e_test.go` are the patterns to copy.

Order, most severe first:

1. **B1 + B2** - validate `traceId` against `^[A-Za-z0-9_-]{1,64}$` in
   `internal/middlewares/log.go:21` (fall back to a generated UUID when the header does
   not match), and again in `TraceHelper.Save`/`Load` as defence in depth. Put
   `/api/trace/:id` behind an admin middleware. Extend `ExcludedHeaders`
   (`response.go:157`) to `stream-secret-key`, `Admin-Api-Key`, `Cookie`, `Set-Cookie`,
   `Proxy-Authorization` - or better, invert it to an allowlist.
   Test: request with a traversal header, assert no file outside `trace_data`; anonymous
   `GET /api/trace/<id>` asserts 401; a captured trace asserts the secret header absent.
2. **B3** - un-nest the `OrderMap` check in `statistic.controller.go:400-408` so
   `order_by` is validated independently of `sort_by`. Add the missing `SortBy` default in
   the `GetUserBreakdownMetricsInWatchInfo` `SumOthers` branch
   (`statistic.repository.go:906`). Test: `POST` with `sort_by` omitted and a quoted
   `order_by`, assert 400.
3. **B4** - validate `filter_by.countries` against an allowlist (ISO-3166 alpha-2), and
   apply the existing `ValidDeviceTypes` check on the breakdown and timeseries handlers
   too. Replace the hand-rolled quoting in `models.MetricFilter.BuildQuery()`
   (`internal/models/action.go:113`) with placeholder args threaded through
   `Table`/`Raw`. Test: a country value containing `'` returns 400 and never reaches SQL.
4. **B5** - require `len(fields) == 2` at `auth_middleware.go:143` and `:170`, matching
   `:110`. Test: `Authorization: Bearer` (no token) returns 401, not 500.
5. **B14 + B15 + B16 + B13** - collapse the two duplicated API-key branches into one
   `validateApiKey(ctx, apiKey, apiSecret)` helper so expiry is defined once; downgrade
   the `UpdateApiKeyLastRequestedAt` failure to a log; return 401 for an unparseable
   `user_id`; route every middleware failure through `ResponseError` rather than
   returning a raw `*echo.HTTPError`.
6. **B6 + B7 + B8 + B9 + B10** - the mechanical `response`/`trace` fixes: move the
   `return err` inside the `IsNotExist` branch; invert the body-capture condition and use
   `io.ReadAll` on a `LimitReader`; accumulate `Total` in `Save` and use `>=`/`||`
   correctly in `Clean`, handling the `Info()` error in the comparator; `io.ReadAll` in
   `Load`; nil-guard both `reflect.ValueOf` calls. Raise the ticker interval from 1s.
7. **B26** - add `syscall.SIGTERM` to `signal.NotifyContext`, pass the signal context to
   the payment watcher and cron, and replace the bare `return` at `main.go:127` with a
   path that still runs shutdown.
8. **B23 + B24** - `viper.AutomaticEnv()` so real environment variables override the
   file; propagate the underlying error into both panic messages; run the existing
   `validate` tags over `AppConfig` after unmarshal.
9. **B12 + B17 + B27 + B28** - the tidy-ups: correct `ResponseFailMessage`, delete
   `internal/utils/errors/`, remove the duplicate constructions in `init.go`, bump
   `go.mod` to match the CI toolchain.

**B18, B25** are recorded as follow-up: both need the separate debug listener and the
config-ownership change that only the fuller restructure provides.

---

## Critical files

| file | change |
| --- | --- |
| `internal/utils/apierr/{apierr,reason,trace}.go` | new, ported from template |
| `internal/utils/apierr/{policy,apierr,readme}_test.go` | new guardrails |
| `internal/utils/response/errorhandler.go`, `message.go` | new; the single `HTTPErrorHandler` and `humanize` |
| `internal/utils/response/{response,errors,trace}.go` | envelope fields, deprecated shim, B6-B12 |
| `internal/utils/version/version.go` + test | new, ported from template |
| `internal/middlewares/auth_middleware.go` | B5, B13-B16; de-duplicate the two key branches |
| `internal/middlewares/log.go` | B1 trace-id validation |
| `internal/controllers/statistic.controller.go` | B3, B4 validation |
| `internal/models/action.go` | B4 `BuildQuery` parameterization |
| `pkg/v1/repositories/statistic.repository.go` | B3 `:906` default; placeholder args |
| `internal/routes/trace.route.go` | B2 auth |
| `cmd/http/{main,init}.go` | error handler registration, B26, B27 |
| `internal/config/config.go` | B23, B24 |
| `Makefile`, `.gitlab-ci.yml`, `go.mod` | B19-B22, B28 |
| `docs/spec_test.go` | new guardrail |

---

## Verification

Each step must be green before the next starts.

```sh
# build must still work under the tag the linter assumes
go build -tags purego ./...
go vet  -tags purego ./...

# the new default
make test                     # go test -race -tags purego ./...
make lint                     # golangci-lint v2.12.2, allow_failure: false in CI
make lint-verify

# the new guardrails, individually
go test -race ./internal/utils/apierr/...      # reason policy + README recipes
go test -race ./internal/utils/response/...    # humanize, 5xx does not leak
go test -race ./docs/...                       # operationId / @Tags / @Failure 500

# the spec must match the handlers
make swagger-verify

# the version stamp must actually land
make build && ./bin/api version
```

End-to-end, against a running API (`docker compose --profile vod up -d postgres redis
rabbitmq`, then `APP_ENV=debug ./bin/api` - note the Slack-token boot blocker recorded in
the project memory note `local-dev-env` still applies):

```sh
# B5 - must be 401, not 500, and must log no stack
curl -si -H 'Authorization: Bearer' localhost:$PORT/api/videos | head -1

# B1/B2 - must not write outside trace_data, and must not serve anonymously
curl -si -H 'stream-trace-id: ../../pwned' localhost:$PORT/api/videos >/dev/null
find . -name 'pwned.json' -not -path './trace_data/*'      # expect: nothing
curl -si localhost:$PORT/api/trace/<id> | head -1          # expect: 401

# B3 - must be 400
curl -si -X POST -H "$APIKEY_HEADERS" -d '{"from":0,"to":1,"order_by":"asc; select 1"}' \
  localhost:$PORT/api/analytics/metrics/bucket/view/media_id | head -1

# B11 - a 5xx body must carry the reason slug and no driver text
curl -s localhost:$PORT/api/<a-route-forced-to-500> | jq .
```

The `docs/spec_test.go` baseline file is expected to shrink over time; it is not expected
to be empty at the end of this change.
