---
type: reference
tags: [testing, backend, storage, security, step-12, verification]
created: 2026-07-16
agent: main
---

Backend Step 12 originally exposed object-path traversal handling in `apps/api/internal/storage/storage.go`. On 2026-07-16 the user approved rejecting raw `.` and `..` segments before normalization; that production fix was applied and its focused storage test passes.

The requested PostgreSQL/Redis Testcontainers integration suites now cover OTP/JWT rotation, role guards, reservation conflicts and expiry, local and MoMo payment validation/idempotency, exactly-once ticket creation, storage safety, and outbox dispatch. `go mod tidy` and `gofmt` completed.

The user approved the standard recursive pattern `go test ./...`. That run confirmed the storage fix and most suites, then exposed the separate [[backend-step-12-reservation-expiry-blocker]].
