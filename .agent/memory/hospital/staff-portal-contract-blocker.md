---
type: fact
tags: [web, staff-portal, api, backend, step-10]
created: 2026-07-16
agent: main
---

The Step 10 backend prerequisite is resolved. `apps/api/internal/staff` implements the dashboard, queue, HomeCare, laboratory, payment, nurse discovery, and demo reset routes declared in `apps/api/api/openapi.yaml`, including `POST /v1/staff/queue/{appointmentId}/status` for `completed` and `absent`.

Queue actions require the `doctor` role, HomeCare actions require `dispatcher`, laboratory actions require `laboratory`, and dashboard, payment visibility, and reset allow all staff roles. Mutations use database transactions and locks, enforce ordered state transitions, and create patient-addressed outbox events. Demo reset clears transactional and clinical state while preserving catalogue data and seeded identities.

`GET /v1/staff/nurses?available=true` is dispatcher-only and returns real nurse UUIDs with name, qualification, hospital ID, image URL, availability, latitude, and longitude. The `available` boolean filter is optional, allowing dispatch views to request either assignable nurses or the full nurse roster.

**How to apply:** Step 10 can use the nurse endpoint to populate assignment controls and API-backed map data, then pass the returned UUID to `POST /v1/staff/homecare/{bookingId}/assign`.
