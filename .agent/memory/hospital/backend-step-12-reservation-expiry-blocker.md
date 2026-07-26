---
type: reference
tags: [testing, backend, reservation, expiry, database, step-12]
created: 2026-07-16
agent: main
---

Backend Step 12 full verification exposed a reservation release blocker. The expiry processor correctly marked a held reservation `expired`, its pending payment `expired`, and its appointment `cancelled`, but a replacement reservation for the same slot rolled back when appointment creation hit PostgreSQL constraint `appointments_slot_id_key`.

The user approved removing global uniqueness while retaining lookup indexing. Migration `00003_allow_appointment_slot_rebooking.sql` drops `appointments_slot_id_key` and adds `appointments_slot_id_idx`; the GORM model now uses a normal index tag. Migration version 3 was applied to local PostgreSQL on 2026-07-16.

The focused expiry/rebooking regression, `go test ./...`, and `go build ./...` all pass. Backend Step 12 coverage now includes OTP/JWT rotation, role guards, reservation conflicts and expiry/rebooking, MoMo and local payment outcomes and validation, callback idempotency, exactly-once ticket creation, storage path safety, and PostgreSQL-to-Redis outbox dispatch.
