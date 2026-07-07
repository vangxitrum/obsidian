# parser-service: gem + ai_task raw-event port

- **gem => DONE** as `gem` source (migration 087). GEM = Global Energy Monitor. Backend gem has 2 ingest paths (both admin file uploads, no poller): (1) GeoJSON pipeline upload -> energy_pipeline_records; (2) Excel .xlsx extraction upload -> field_level_main/production tables. Only the **GeoJSON pipeline** path fits the raw-decode-and-save pattern, so that is what was ported. Table `energy_pipeline_records`, conflict key (project_id, fuel). Excel extraction path deliberately excluded (binary xlsx needs excelize dep => go.mod edit forbidden; not a JSON payload).
  - Files: internal/types/gem.go, internal/datasource/gem.go, internal/database/gem.go, migrations/087_energy_pipeline_records.{up,down}.sql
  - Publisher mapping: layer=energy-pipeline, dataset=gem.energy_pipeline, source=gem
  - Register: `.Register(datasource.NewGEMHandler(db, logger))`
- **ai_task => SKIPPED.** Not a datasource. `ai_tasks` is an internal work queue (uuid/source_id/type/status/retry). Backend fetcher polls DB (GetPendingTasks), calls internal AI service client, writes results into OTHER tables (news countries, csg force posture). No external raw HTTP payload, no decodable raw blob => does not fit raw-decode-and-save.
