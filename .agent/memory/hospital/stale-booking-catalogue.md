---
type: reference
tags: [mobile, booking, seed, catalogue]
created: 2026-07-26
agent: main
---

The doctors API only returns doctors with future appointment slots. Because demo slots are generated relative to seed time, an old database can validly return HTTP 200 with an empty doctors array after all slots pass. The mobile `demoApi.catalogue` must therefore treat empty successful catalogue responses like unavailable demo data and retain seeded specialties, doctors, and HomeCare services. Appointment booking likewise uses generated demo times when the slots API succeeds but has no available rows, not only when it errors. Regression coverage is in `apps/mobile/__tests__/api.test.ts`.

Run the destructive demo seed to restore the real API booking path when dated slots expire. This resets sessions and mutable demo records, so users must log in again with OTP `123456`.
