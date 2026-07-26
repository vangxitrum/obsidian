---
type: reference
tags: [patient-api, mobile, ownership, openapi, runtime-integration]
created: 2026-07-16
agent: main
---

The patient runtime integration is implemented across `apps/api`, `apps/api/api/openapi.yaml`, generated Go/TypeScript clients, and `apps/mobile`. Authenticated patient handlers now serve family profiles, specialties, filtered doctors, future slots, HomeCare services, appointment ticket/check-in/queue, owned HomeCare booking/result/vaccination data, and notifications. Every patient-specific read joins through the owning reservation or filters by `user_id`.

Reservation GET responses now expose optional `appointmentId` or `homeCareBookingId`. Mobile API payment completion fetches the confirmed reservation and persists that real entity UUID before showing success or navigating. API login hydrates real family UUIDs and catalogue data; appointment slots and HomeCare dates are future-relative rather than fixed July dates.

No migration was needed because the ownership and clinical-result schema already existed. Verification on 2026-07-16 passed authenticated seeded API E2E, `go test ./...`, `go build ./...`, generated-client lint/typecheck, mobile lint/typecheck/Jest (7 tests), and Expo iOS/Android exports under Node 22 with pnpm 11.13.0. The deterministic seed was restored after E2E.

Offline behavior intentionally retains seeded catalogue/profile IDs, locally generated demo reservation IDs, simulated ticket/queue/nurse/result/vaccination content, and local in-app notifications when API requests are unavailable.

Catalogue handlers must map GORM `Specialty` and `HomeCareService` models to explicit lower-camel API responses. Returning those models directly serializes uppercase Go field names such as `ID`, leaving mobile `item.id` undefined and causing React duplicate-key warnings. Regression coverage lives in `apps/api/internal/patient/handler_test.go`.
