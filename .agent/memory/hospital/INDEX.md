# hospital - memory index

- [[prototype-scope]] - Demo scope selects two focused patient journeys, a staff portal, simulated shared state, and Vietnamese-first accessible UI.
- [[implementation-stack]] - Implemented monorepo uses Expo, Next.js, Go/Echo/GORM, PostgreSQL/Supabase adapters, Redis, MinIO, WebSocket, and MoMo adapters.
- [[mobile-step-9]] - Expo patient application is implemented with both demo journeys; mobile lint and typecheck pass.
- [[staff-portal-contract-blocker]] - Step 10 staff backend prerequisites are complete, including dispatcher nurse discovery with real IDs and coordinates.
- [[web-step-10]] - Staff portal and local payment simulator are implemented and all web checks pass; fresh-stack runtime still needs migration and seed commands.
- [[demo-seed-step-11]] - Deterministic GORM demo seed and pinned Goose migration/reset commands make a fresh database demo-ready.
- [[web-step-12-tests]] - Web Step 12 is complete: static checks, 7 unit tests, and 8 real-API Playwright tests pass under Node 22.
- [[mobile-step-12-tests]] - Mobile Jest/RNTL coverage and both Maestro scripts are implemented; all runnable mobile checks pass.
- [[backend-step-12-storage-blocker]] - Backend Step 12 storage traversal fix and focused regression coverage are complete.
- [[backend-step-12-reservation-expiry-blocker]] - Migration 3 resolves expired-slot rebooking; all backend Step 12 tests and builds pass.
- [[patient-runtime-integration]] - Authenticated owned patient APIs and mobile real-ID payment discovery are integrated and fully verified.
- [[demo-tunnel-endpoints]] - Public Expo, API/WebSocket, and web/payment tunnel hostnames for cross-network phone testing.
- [[local-docker-runtime]] - All local application processes run as detached Docker containers without tmux.
- [[redis-aof-recovery]] - Recover the local Redis volume after an unclean shutdown leaves a truncated AOF tail.
- [[demo-queue-counter]] - Patient queue demo advances a shared current number every randomized 1-3 minutes with an offline mobile fallback.
- [[stale-booking-catalogue]] - Empty successful doctor/slot responses from expired demo seed data must activate mobile booking fallbacks.
- [[expanded-clinical-catalogue]] - API and mobile now share 9 specialties, 20 imageless doctors, and 12 priced HomeCare services.
