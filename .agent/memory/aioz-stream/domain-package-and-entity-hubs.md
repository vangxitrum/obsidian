---
type: decision
tags: [architecture, go, packages, vertical-slice, import-cycles]
created: 2026-09-09
agent: main
---

`internal/models` was renamed to `internal/domain` and kept deliberately as a
shared leaf package. Only single-owner entities were moved out into their slices.

**Why keep a shared package at all:** the fan-in is real domain coupling, not
leftover laziness. Measured references from `internal/app/*` slices:

| file | slices |
|---|---|
| `authentication_info.go` | 14 |
| `error.go` | 13 |
| `user.go` | 12 |
| `video.go` | 10 |
| `usage.go` | 8 |

Ten slices talk about videos because ten slices genuinely talk about videos.
Moving `Media` into `internal/app/media` would make 10 slices import `media` and
create a cycle the first time `media` needed anything back. A shared leaf package
is the correct answer for these; the rename just makes it say so.

**The blocking constraint — entity hubs.** A child entity cannot leave `domain`
while a struct that stays in `domain` still embeds it, or `domain` would have to
import the slice (cycle). The GORM association graph pins these chains:

- `domain.User` → `EditorProject` → `HighlightMedia` → `HighlightChunk` / `HighlightClip` / `HighlightManifest`
- `domain.Media` → `MediaCaption`, `MediaChapter`, `Part`, `Watermark`, `MediaWatermark`
- `domain.User` → `ExclusiveCode`, `WalletConnection`

So the whole highlight cluster and the caption/chapter/part/watermark cluster are
immovable until `User` and `Media` themselves are restructured. **Check this
before planning any further move** — grep `internal/domain/*.go` for the symbol;
the compiler is the only reliable oracle (a hand-rolled grep check gave a false
"safe" and cost a full revert).

Two other pins that are not hub-related:
- `api_key.go` — `store.UseCase` holds `ApiKeyRepository`, so moving it needs `UseCase` restructured.
- `live_stream_key.go`, `variable_sort.go` — blocked only by `internal/utils/response/allowed_test.go`, which is `package response` (internal). Making it `package response_test` would unblock both.
- `report_content.go` — `internal/utils/message/message.go` uses it; `report` imports `message`.
- `format.go`, `stream.go` — `internal/core` uses them; `media` imports `core`.

**DAO was considered and rejected.** There were zero import cycles to begin with;
the existing dependency inversion (slice owns the entity + repository interface,
`pkg/v1/repositories` implements it and imports the slice) is what prevents them.
A DAO layer would only decouple persistence schema from domain type — it cannot
fix domain coupling, and costs ~50 entity/mapper pairs plus the loss of GORM
`Preload` / `OnConflict` / partial-update ergonomics. If the four-jobs-one-struct
problem (GORM table + JSON response + Swagger type + domain object) bites, split
the *response* types into slice-local `dto.go` instead.

See [[models-horizontal-split]] for the earlier layer-wise split and
[[slice-migration-state]] for overall slice migration progress.
