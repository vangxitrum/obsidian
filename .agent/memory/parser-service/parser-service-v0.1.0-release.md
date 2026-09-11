# parser-service v0.1.0 - first release (2026-09-07)

First tagged release cut from the whole history (no prior tag; 63 commits / 57 non-merge,
566 files, 101k insertions from `3825920` initial commit). Note NOT yet tagged in git -
the note is generated from `root..HEAD`, user tags when satisfied.

Published:
- repo: `.gitlab/releases/v0.1.0.md` + `v0.1.0-review.md` (dir created by this release)
- vault: `Projects/parser-service/releases/` + new `Projects/parser-service/INDEX.md`

## What the service actually is (as of v0.1.0)
Consumer half of AIOZ Map: subscribes to crawler-service RabbitMQ raw-document events,
parses into Postgres, serves ~70 gated GET endpoints plus auth/API keys/layouts/alerts.
Two binaries: `cmd/server`, `cmd/consumer`. 20 handlers registered in
`internal/app/process.go`. 3 migrations: baseline, flight tracking, aircraft detail cache.

Structure note: there is NO `internal/datasource/` package here (that is crawler-service).
Parser handlers live in `internal/consumer/`, DB writers in `internal/database/`,
HTTP in `internal/api/<feature>/` (dto/endpoint/service/store).

Feature gates default CLOSED: `api.disabled: true` + `api.allowed`, `broker.disabled: true`
+ `broker.allowed_event_types`, `alert.chore_disabled: true`. Gated-off routes are never
registered and are stripped from the served Swagger doc.

## Release verdict: fix-first
Review scoped to the aircraft-detail-panel branch (`5b8d6cb..243316b`) - the full range is
the whole repo and not reviewable in one pass. 12 findings, 3 blocking:
1. `store.go:368` - `flight_date` is DATE, compared against a Go `time.Time` bound as
   timestamptz with no `TimeZone` pinned in the DSN. On a non-UTC server the equality never
   matches -> "reread flight occurrence" -> no flight occurrence ever persists ->
   `LastFlightsSyncAt` stays nil forever. Hardest bug in the release.
2. `service.go:113` - `position != nil &&` guard defeats the staleness 404 gate; unknown
   icao24 falls through to two paid AeroDataBox calls. `/api/v1` has no middleware, so it
   is anonymous-reachable. `isPositionStale(nil, now)` already returns true.
3. `endpoint.go:85` - `limit=-1` skips the clamp and `findLatestByBBox` only applies Limit
   when `> 0`: unbounded unauthenticated full-table read.

Also: singleflight captures first caller's ctx (cancel poisons sharers), empty flight lists
are never negative-cached so the 8h TTL never applies, `tx.Save` blanks omitted columns,
`findOrCreateFlight` under-keys on NULL airport IDs.

## Corrections to older memory
- [[router-auth-gating-gap]] is largely FIXED: user/apikey/layout/dashboard-layouts/alert
  groups all carry `e.AuthMiddleware` now. One public write remains:
  `PUT /api/v1/weather/dataset/layers/availability` (under the weather gate, no auth).

Related: [[shared-aioz-logger]], [[hermes-kiro-credentials-dead]]
