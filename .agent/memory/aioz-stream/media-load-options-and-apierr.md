---
type: decision
tags: [performance, gorm, errors, apierr, architecture]
created: 2026-09-09
agent: main
---

Two changes shipped together on `refactor/sync-with-template-source`.

## 1. Media load options (the over-fetching fix)

`GetMediaById` issued **18 Preloads = 19 queries** on every call, across 28 call
sites, because `domain.Media` has 11 associations and there was one load shape
for all callers. `GetUserMediaById` was 16.

`store.MediaLoad` now says which associations a caller needs;
`store.FullMediaLoad()` is the old shape. `applyMediaLoad` in
`pkg/v1/repositories/video.repository.go` is the single place association names
are spelled, so the two by-id queries cannot drift.

Each slice declares its own load next to its port, with a comment saying what it
reads. Measured, not guessed:

| slice | load | why |
|---|---|---|
| `report` | `{}` | only `GetHlsPlayerUrl` -> Id, Secret |
| `summary` | `{}` | only `IsDeleted` -> Status |
| `live` | `{}` | only `Status` / `IsDone` |
| `player` | `{}` | only `Id`, `PlayerThemeId` |
| `statistic` | `{Format:true}` / `{}` | `GetMediaDuration` reads `Format.Duration`; the other site reads `UserId` |

That is 19 queries -> 1 (or 2) at those sites.

**Deliberately left on `FullMediaLoad()`:** `internal/app/live/video.go` and the
32 sites inside the `media` slice. Those are read-modify-writes ending in
`UpdateMedia` -> gorm `Save`, and `Save` persists whatever associations the
struct carries. Narrowing them needs a test proving `Save` leaves unloaded
associations alone. The comment in `live/video.go` says so.

**How to add a caller:** never pick a load in the service - `MediaService.GetMediaById`
passes the caller's load straight through. Guessing at the service layer is what
made every read cost 19 queries.

## 2. apierr everywhere

396 legacy `response.New*Error` sites converted to `apierr` (348 in `media`,
which had zero before, plus 48 in `live`, `usage`, `middlewares/auth`,
`utils/validate`, `middlewares`). The three sentinel vars
(`UnauthorizedError`, `CodeExpiredError`, `InvalidSignatureError`) and six dead
constructors are gone; `internal/utils/response/errors.go` was deleted.

The apierr -> echo mapping already existed and was not touched: `classify` +
`renderError` in `internal/utils/response/response.go`, registered as echo's
`HTTPErrorHandler`. One envelope, whether a handler returns the error or calls
`ResponseError`.

Reason slugs were generated from the message literal where there was one
(`title-required`, `signature-invalid`) and from the enclosing function
otherwise (`create-media-object-failed`). They are consistent, not hand-crafted -
worth refining if a client starts branching on a specific one.

New error tags added: `media`, `auth`, `validate`, `middlewares`.

**Verified:** build, vet, 42/42 packages pass, `make lint` 0 issues. Note the
linter needs `goimports -local 10.0.0.50/tuan.quang.tran/vms-v2` - plain
`goimports` gets the import grouping wrong and CI fails.

See [[domain-package-and-entity-hubs]] for why DAO was rejected as the fix for
this - it would not have reduced query count.
