---
type: reference
tags: [database, seed, goose, demo-data, step-11]
created: 2026-07-16
agent: main
---

Step 11 added `apps/api/cmd/seed/main.go`, a transactional GORM seed that truncates all application tables without touching `goose_db_version` and recreates fictional Vietnamese demo data with stable UUIDs. Relative dates keep the operational dashboard populated, and the frozen bcrypt hash authenticates all staff accounts with `Demo123!`. Seeded local payment URLs derive from `WEB_ORIGIN`, so public deployments do not persist localhost links.

Root commands are `db:migrate`, `db:seed`, `db:reseed`, and `db:reset`; Goose is pinned to `v3.26.0` and every Go invocation uses `go -C apps/api`.

Verification on 2026-07-16 ran migration plus consecutive reset/reseed operations, preserving migration version 2 and identical counts. Go tests/builds and the complete pnpm workspace typecheck passed. Real API authentication and role-scoped dashboard, queue, HomeCare, nurse, and laboratory reads returned populated data.

On 2026-07-21, stale demo slots caused `/v1/doctors` to have no results; GORM `Scan` serialized its nil slice as JSON `null`, which crashed mobile Find/search calls using `.find()` and `.filter()`. `patient.Service.Doctors` now initializes a non-nil empty slice, with regression coverage. The live demo was reseeded to restore future availability; because slot dates are relative to seed time, reseed the demo when its schedule ages out.

On 2026-07-22, the old four-specialty fictional catalog was replaced in both the backend seed and mobile fallback with 5 specialties and 12 user-supplied Tâm Anh doctor profiles: Da liễu - Thẩm mỹ da (2), Tai Mũi Họng (3), Cơ Xương Khớp (2), Sản Phụ khoa (2), and Ung bướu (3). IDs are aligned between backend and mobile, every doctor has a future slot, and the live public demo was reseeded. Public API counts, name search, and the public Expo bundle were verified.
