---
type: reference
tags: [web, testing, vitest, playwright, step-12]
created: 2026-07-16
agent: main
---

Step 12 web testing is implemented under `apps/web`: Vitest and Testing Library cover all staff payment status labels/classes and all five local payment simulator outcome payloads plus API rejection rendering. Playwright is configured for serial real-API tests with deterministic `db:reset`, real API/Next web servers, all three staff role workspaces, doctor queue call/complete/absent, seeded HomeCare assignment and all forward statuses, laboratory publication, payment details, and destructive demo reset last.

Verification completed on 2026-07-16 under transient Node v22.23.1: web ESLint, TypeScript typecheck, all 7 Vitest tests, and all 8 Playwright Chromium tests pass. Browser coverage uses the real API and deterministic Go seed and verifies all three role workspaces, doctor queue call/complete/absent, seeded HomeCare assignment and complete status timeline, laboratory publication, payment details, and final demo reset. Trace diagnosis also found and fixed a production defect: backend `bookingPatient` scanned a PostgreSQL UUID directly into byte-array `uuid.UUID`, causing HomeCare status updates to return 404; a one-field struct scan resolves it.

**How to apply:** Run the web checks through transient Node 22 when system Node differs. Playwright expects PostgreSQL on 5432 and Redis on 6380, starts API on 8180 and Next on 3100, reseeds local data directly through Go, and ends by clearing operational demo data through the UI reset.
