---
title: AGM BE Self-Review test suite (113 TCs) - coverage and the 4 API gaps
type: project
updated: 2026-08-21
---

# AGM `BE - Conduct Self-Review & Functional Testing` - test coverage

Done 2026-08-21. Eight subtasks (AGM-97/98/99/101/102/103/104/105) carry a
**Test Cases** checklist in their Jira description; DoD item 4 requires the
results posted as a ticket comment. All 8 comments posted.

Result: **39 covered, 10 partial, 57 blocked, 7 FE** of 113.
Canonical matrix: `parser-service/docs/TEST_CASES.md` (crawler view in
`crawler-service/docs/TEST_CASES.md`).

## Convention

Tests are named `TestAGM<ticket>_TC<nn>_<what>`, so `go test ./... -run TestAGM98 -v`
prints a ticket's checklist. 41 such tests in each repo.

## The 4 API gaps that block 59 TCs

Verified absent by grep across parser-service, crawler-service AND backend:

1. **No earthquake detail endpoint** (`GET /weather/earthquake/:id`). Only list +
   `/:id/shakemap`. Data is already on the list DTO, so this is small API work.
2. **No fire detail endpoint** - and it is NOT a missing handler. FIRMS is a
   *thermal hotspot* feed (one row per satellite pixel: FRP, brightness,
   confidence). No fire id, name, acreage, status, perimeter. AGM-11 needs an
   incident-level upstream (NIFC/IRWIN), not a new route.
3. **No volcano detail endpoint, and `VolcanoDTO` has no `id` field at all** -
   `vnum` is the only stable key. Also the shipped alert set is
   NORMAL/ADVISORY/WATCH; AGM-29's four-state matrix (incl. "erupting") does not
   exist in the data.
4. **No weather-code-by-location-names endpoint.** `grep location_names` = 0 hits;
   everything is keyed on `location_ids`. Dead `LocationRequest`/`LocationResponse`
   trio in `weather_forecast_daily/dto.go` + `location.Service.SearchProvince`
   would do most of the work.

## Defects found

- **AGM-98 TC05/TC07** (FIXED 2026-08-21): `mag_from`/`mag_to` and
  `start_date`/`end_date` were each validated in isolation, never compared, so
  `?mag_from=7&mag_to=4` returned 200 with an empty list instead of 4xx. Both
  pairs are now compared in `internal/api/earthquake/endpoint.go` and rejected
  with a 400 **before** the store is touched.
  Boundary decisions: equal bounds stay valid (both SQL predicates are
  inclusive), a lone bound stays valid, and dates are compared as parsed
  *instants* not wall-clock text (an RFC3339 offset cannot smuggle an inverted
  window through). Swagger annotations updated + regenerated.
- **volcano vs earthquake inconsistency**: `alert_levels=NORMAL,` (trailing comma)
  400s in volcano because its split loop does not skip blanks; earthquake's does.
- `GetShakeMap` has no swagger annotations -> missing from the generated spec.

## Test seam added (this is the enabling change)

`Endpoint -> Service -> Store` was concrete at every level, so handler tests could
only reach validation branches. Added a narrow unexported store interface next to
each `Service`, copying the existing `map_extract.provinceSearcher` pattern:
`earthquakeStore`, `firmsStore`, `volcanoStore`. `NewService` still accepts
`*Store` (satisfies the interface) so `internal/app/server.go` is untouched.

No new deps - repo suite is bare `testing`, no testify/sqlmock/testcontainers.

Related: [[be-api-dod-template]], [[agm-crawler-test-coverage]]

## 2026-08-24 merge from origin - fallout and an unresolved schema question

Merged `origin/refactor/map-layer-v1` (0a63e7f) into this branch. 5 textual
conflicts (docs.go/swagger.json/swagger.yaml/go.mod/go.sum), all auto-resolved
cleanly by git (no line overlap with my aioz-log addition). Real breakage was
semantic, not textual:

- **firmsStore interface went stale.** Upstream's earlier `86f032d` refactor
  (already on origin, landed via this merge) replaced GetForecastURL's direct
  producerclient call with 4 new Store methods
  (GetDatasourceDocumentByDate/GetLatestDatasourceDocumentFromDate/
  GetLatestDatasourceDocument/ListDatasourceDatesFrom) and changed
  GetForecastDates to take a ctx and look up REAL published dates instead of a
  hardcoded per-provider day count (ecmwf=9/meteofrance=3/nasa=6, gone).
  Extended the interface + fake to match; rewrote TestAGM101_TC01/02/04/05
  entirely around the new DB-backed lookup instead of asserting fixed day
  counts. This is actually a coverage IMPROVEMENT (previously-BLOCKED TC02
  bounds-half is now closer to real).
- **GetShakeMap endpoint REMOVED.** `61b508b "Remove weather forecast and ISS
  position handling code"` (an MCP-tool cleanup, message never mentions
  ShakeMap - looks incidental) deleted `GET /weather/earthquake/:id/shakemap`
  and `Endpoint.GetShakeMap`. Deleted the two tests that hit it
  (TestAGM98_TC14/15); reverted AGM-98 from 17/18 covered back to 15/18 - TC14
  and TC15 are BLOCKED again. The shakemap-in-list-embedding path (List() ->
  Service.GetShakeMaps -> Store.GetShakeMapsByEventIDs) was NOT touched and
  stays covered via TC13.
- **8 pre-existing lint findings from upstream code** (never touched by me):
  2x errorlint (`err != gorm.ErrRecordNotFound` -> `errors.Is`, which then
  exposed a real govet nilness tautology - `err != nil &&` after an already-
  `err == nil` early return - simplified both), 4x goimports (local-prefix
  grouping, `goimports -local gitlab.internal/aioz-map/parser-service`), 1x
  ineffassign (dead `closeResources` reassignment in server.go - shutdown
  actually goes through separate `closeDB`/`closeGRPC` fields on ServerApp, so
  this was truly dead, not a masked resource leak - deleted), 4x unused
  (orphaned DTO mappers in location/dto.go + dead `earthquakeRawResponse` type
  in usgs_earthquake.go, left behind by upstream's endpoint deletions -
  deleted). Back to 0 issues.

### UNRESOLVED, flagged to Tuan, not fixed

`0a63e7f "Refactor database migrations..."` squashed everything into a single
`001_baseline_schema.up.sql` and this baseline **does not create
earthquake_records, volcano_records, fire_records, or shakemap_records**.
Every line of Go code that queries those 4 tables (internal/database/{earthquake,
volcano,firms,shakemap}.go + the 3 api/*/store.go read sides) is completely
untouched. Against a freshly-migrated DB, every AGM-98/99/101/102/103/104/105
endpoint fails at the SQL layer. Ambiguous whether accidental (baseline
generated from a stale dump) or intentional (would be odd - no Go code was
touched to match, unlike the shakemap ROUTE removal above, which WAS paired
with code changes). Did not restore the migrations or delete the Go code
without asking - this changes runtime behavior materially either direction.

## Second rebase, 2026-08-24 (same day, later) - alert/layout re-added

Branch was already merged+pushed (efa8a0f on top of 0a63e7f) when a further
rebase landed on origin (no conflict markers, tree already clean/in-sync) that
reintroduced `internal/api/alert` + `internal/api/layout` (deleted by the
EARLIER 61b508b cleanup, now brought back by `a058cc7 "feat: implement layout
service with CRUD operations and integrate alert service"`). Two real breaks,
same pattern as before - silent semantic loss during rebase's line-based merge,
zero textual conflicts:

1. **firmsStore interface stale again.** A further store.go refactor added
   `UpsertDatasourceDocument` on Service, built from 3 new Store primitives
   (FindDatasourceDocument/CreateDatasourceDocument/UpdateDatasourceDocumentFields).
   Extended interface + fake. **These 3 have no caller anywhere in the codebase**
   (not wired to any endpoint/consumer) - only Store's OWN separate
   `UpsertDatasourceDocument(doc)` gorm-level method is actually used, and it
   isn't reached through this interface. No TC exists for them; fake methods
   exist purely to satisfy the interface.
2. **alert.Endpoint/layout.Endpoint left nil.** router.go unconditionally
   registers `/alert`, `/dashboard-layouts`, `/layout` routes calling
   `e.Alert.*`/`e.Layout.*`, but server.go's EndpointSet{} literal never set
   those fields (would nil-panic on first real request) - AND server.go still
   imported `alert`/`alertchore`/`alertprocessor`/`layout` with the actual
   construction code missing (classic silent-drop from a line-based rebase
   merge). Wired `alertEp`/`layoutEp` following the exact pattern of the other
   ~30 endpoints in NewServerApp (Store->Service->Endpoint, all *slog.Logger +
   *gorm.DB). Did NOT wire `alertchore`/`alertprocessor` (cleanup cron, outbox
   draining, per-source processors) - nothing in the codebase constructs them
   ANYWHERE (not process.go, not server.go), so there's no existing call
   pattern to safely mirror; dropped those 2 now-genuinely-unused imports
   rather than fabricate cron expressions / alert-type strings / handler funcs.
   Flagging: **outbox/chore draining for alerts is unwired** - real gap, needs
   the feature's original author.
3. Also cleaned a 3rd recurrence of the SAME dead-`closeResources` pattern in
   server.go (rebase reintroduced the pre-fix declaration while keeping the
   post-fix direct `closeDB()` call on the error path - inconsistent partial
   merge of both my earlier fix and the original code). Deleted again.

Verified: alert/layout DO have backing tables in 001_baseline_schema.up.sql
(`alerts`, `alert`, `alert_outbox`, `layouts` x2 (duplicate!), `widgets`) -
unlike earthquake/volcano/fire/shakemap. Not the same class of problem.

Build/vet/test/lint all clean after. 3 files touched:
internal/api/firms/{service.go,endpoint_test.go}, internal/app/server.go.

**Pattern to expect on any future rebase of this branch:** check for (a)
stale `firmsStore`/other narrowed interfaces vs whatever store.go now has,
(b) EndpointSet{} literal missing fields for anything router.go references,
(c) `closeResources`-style dead-var reintroduction in server.go. All three
have now recurred across 2 separate rebases with zero git conflict markers.
