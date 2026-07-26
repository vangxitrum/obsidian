---
type: decision
tags: [implementation, architecture, go, expo, nextjs, supabase, payment]
created: 2026-07-16
agent: main
---

The hospital prototype is implemented as a pnpm/Turborepo monorepo in `/home/tuan/personal/hospital`.

- `apps/mobile`: Expo 54, React Native 0.81, and React 19.1 patient app, package ID `vn.tamanh.demo`. SDK 54 intentionally matches the Google Play Expo Go 54 client.
- `apps/web`: Next.js staff portal and local payment simulator.
- `apps/api`: Go 1.25 modular monolith using Echo, GORM, Goose, Asynq, Redis Pub/Sub, and WebSocket.
- Local infrastructure: PostgreSQL on 5432, project Redis on 6380, and MinIO on 9000/9001.
- Managed adapters: Supabase PostgreSQL and Storage, Railway API/worker, Vercel web, Expo EAS.
- Payment: local simulator and MoMo Test adapter; ten-minute holds and verified IPN confirmation.

Stable credentials are patient `0901234567` with OTP `123456`; staff users `bacsi.demo`, `dieuphoi.demo`, and `xetnghiem.demo` with password `Demo123!`.

**Why:** This preserves one TypeScript frontend ecosystem while using Go for concurrency-sensitive reservation, payment, job, and realtime workflows.

**How to apply:** Use Node 22 and pnpm 11.13.0. Run `README.md` first-run commands. Never point reset/seed commands at shared or production data. See `docs/DEPLOYMENT.md` for Supabase and MoMo activation.
