---
type: decision
tags: [database, postgres, tunnel, runtime]
created: 2026-07-21
agent: main
---

The Supabase project `rsuvklfxtyipynhfvfai` was initially unavailable. Its API hostname later began resolving, but the saved anon key returns HTTP 401 and both known pooler hosts reject the database tenant. The direct database hostname is IPv6-only and is unreachable from this machine. The locally tunneled app therefore uses the `essay-manager-postgres` Docker container, bound to `127.0.0.1:5436`, with persistent volume `essay-manager-postgres-data`. The ignored `.env.local` overrides `DATABASE_URL`; credentials remain there and are not recorded in memory.

The `writing.tunnel.appdemo.cyou` reverse tunnel forwards to the Next.js development server on port 3002. `next.config.ts` must include that hostname in `allowedDevOrigins` for development assets and HMR.

**Why:** The unavailable remote project caused every home-page query to fail with HTTP 500.

**How to apply:** Ensure the Docker container and Next.js server are running before exposing the tunnel. Existing Prisma migrations have been applied to the local database.

The local database has an `admin` user with role `ADMIN`. Its password is intentionally not stored in memory. Migration `20260721230500_add_user_theme` is required because the initial migration omitted the `User.theme` column used by the current Prisma schema.

The legacy `dev.db` still exists in the repository working directory even though its deletion is staged. Its hash matches the latest Git copy and it contains 100 prompts, one admin user, one essay owned by that user, and one evaluation. This data can be migrated into local PostgreSQL; do not delete or overwrite `dev.db` before recovery. The unreachable Supabase project may have newer data, but no local PostgreSQL dump was found.
