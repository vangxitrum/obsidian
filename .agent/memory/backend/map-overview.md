---
type: fact
tags: [aioz-map, backend, go, gin, postgres, datasource]
created: 2026-07-05
agent: main
---

**aioz-map backend** (aka the "MAP" project). Located at `/home/tuan/work/backend`.
Go service: periodically fetches data from external APIs, persists to PostgreSQL
(GORM + AutoMigrate), exposes via Gin REST API + Swagger.

Repo: `git@gitlab.internal:aioz-map/backend.git`. Default working branch: `develop`.

## Binaries (all share one codebase)
- `api` — `cmd/api/main.go` — Gin HTTP server, REST endpoints + Swagger UI.
- `datasource` — `cmd/datasource/main.go` — background runner, polls external APIs on schedule.
- `mockserver` — `cmd/mockserver/main.go` — serves file-backed fixtures for testing.

## Layout
- `internal/config/` — Viper config loader.
- `internal/database/` — GORM open + AutoMigrate.
- `internal/app/` — dependency wiring (`APIApp`, `DataSourceApp`).
- `internal/api/` — `server.go` (Gin lifecycle), `router.go`, `response/`, `middleware/`.
- `internal/datasource/` — `manager.go` runs DataSources concurrently on intervals.
- `internal/utils/` — `log/` (colorized slog), `lifecycle/` (graceful start/stop).

## Key concepts
- **DataSource** (`datasource/datasource`) — one per external API. Owns token-bucket
  rate limiter, auth (HTTP header or query param), operational status persisted to
  `data_providers` table.
- **DataFeed** (`datasource/datafeed`) — one per logical feed; per-feed poll tracking,
  HTTP/WS execution.

## Data providers wired
- **weather** — OpenWeatherMap (config/model/fetcher/service/endpoint), weather tiles.
- **coinglass** — orderbook, price, and daily-scope feeds: liquidation exchange,
  liquidation history, liquidation heatmap, whale transfer, liquidation order.
  Shared `dailysync/` handles daily-scope persistence (scope lifecycle, backfill).
- earthquake feed, flight feed (seen in recent commits).
