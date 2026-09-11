---
type: fact
tags: [uuid, postgres, gorm, aioz-common]
created: 2026-09-10
agent: main
---

aioz-common `uuid.UUID.Value()` (v0.2.1-0.20260908094255) returns the 16 raw bytes. That works for `uuid` columns, but inserts into legacy `text` id columns fail with `invalid byte sequence for encoding "UTF8": 0xa0 (SQLSTATE 22021)`. First seen on `payment_logs.belongs_to_id`.

The old google/uuid `Value()` returned the string, so gorm AutoMigrate created untagged uuid fields as `text`. video-db has many of these: usage_logs.id/user_id, wallets.user_id, playlist_items.id/next_id/previous_id, payment_logs.belongs_to_id, and others.

The raw-bytes Value also makes gorm infer DataType `bytes` for untagged fields (the string form gives `string`), so AutoMigrate would try to change those columns.

**Fix (in the common package):** `Value()` returns `id.String()`. Keep `Scan` as it is. It landed in aioz-common commit `b5294870c745` (pseudo-version v0.2.1-0.20260910081149-b5294870c745). On 2026-09-10 I verified it end to end: `PaymentRepository.CreatePaymentLog` writes into the real `payment_logs` table and the values read back correctly.

**Gotcha:** `go mod tidy` fails in this repo because the root-owned `postgres_data/` directory (a Docker bind mount) blocks the `all` pattern. Instead, fill in go.sum with `go test -mod=mod -run '^$' ./cmd/... ./internal/... ./pkg/...`.

**Gotcha:** gorm `Raw(...).Scan(&id)`, where `id` is a single `uuid.UUID`, fails with `converting driver.Value type string to a uint8`. gorm treats the `[16]byte` array like a slice of rows. google/uuid had the same problem. Scan into a struct field instead.

**How to verify:** use a `-overlay` repro in `internal/tools/uuidprobe` (build tag `uuidprobe`, env `UUID_PROBE_DSN`, db container `aioz-stream-db` on port 5437, db `video-db`).

Related: [[gosdk-storagehelper]], [[local-dev-env]]
