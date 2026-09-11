# aioz-stream

## Plans

- [2026-07-09: go-sdk-backed StorageHelper implementation (impl + wiring + config)](plans/2026-07-09-gosdk-storagehelper.md)
- [2026-08-25: Wide-input-range VOD support for the worker](plans/2026-08-25-vod-wide-input-support.md) - 32 prioritized gaps in the worker's ffmpeg arg generation blocking wide-range VOD input support; includes the HEVC-on-iOS / Bento4 analysis.
- [[2026-09-03-import-media-from-external-hls-url]] - plan: import media from an external HLS (m3u8) URL by mirroring renditions into storage.
- [[2026-09-07-align-aioz-stream-with-aioz-template]] - contracts-first alignment with the house Go template, plus the bug report and fixes.
- [[2026-09-09-monkit-observability-migration]] - design: replace the prometheus metrics package with aioz-common/stats (monkit), add the debug server, move /metrics and /version off the public port, and drop the Grafana/Loki stack from the repo.
- [[2026-09-09-monkit-observability-migration-plan]] - implementation plan (9 tasks) for the monkit observability migration.
- [[2026-09-10-one-binary-aiozstream]] - merge api/grpc/migrate into one aiozstream binary with --config-dir
- [[2026-09-10-aioz-common-v030-clients]] - draft plan: move Postgres (GORM on the common pgxpool), Redis, RabbitMQ and outbound HTTP to aioz-common v0.3.0, with common Config sections; 4 open decisions.
- [[2026-09-10-upstream-golang-migrate]] - use upstream golang-migrate CLI, drop aiozstream migrate cmd and gorm AutoMigrate
