---
type: reference
tags: [queue, patient-app, demo, api]
created: 2026-07-26
agent: main
---

The patient queue endpoint owns a process-local demo counter per queue date. On the first queue request it starts immediately before the day's first ticket, advances one number after each randomized 1-3 minute interval while the mobile app polls, and treats real staff progress as a floor. It does not mutate ticket statuses. The mobile queue screen prominently renders the shared current number, recalculates a demo wait estimate at two minutes per remaining number, distinguishes the patient's current turn, and runs the same randomized progression locally if the API is unavailable.

The process-local state intentionally resets when the API restarts so each demonstration begins from a useful baseline. Backend timing coverage is in `apps/api/internal/patient/service_test.go`; mobile rendering coverage is in `apps/mobile/__tests__/queue.test.tsx`.
