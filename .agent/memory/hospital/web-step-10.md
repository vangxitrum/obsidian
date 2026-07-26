---
type: reference
tags: [web, staff-portal, nextjs, step-10, verification]
created: 2026-07-16
agent: main
---

Step 10 is implemented in `apps/web` as a responsive Next.js App Router staff portal and local payment simulator. It includes persistent real-API staff login with role selection, role-scoped navigation, dashboard/reset, doctor queue actions, HomeCare nurse assignment/status/map, laboratory publishing, payment list/detail, reconnecting WebSocket updates with REST refresh, and `/payment/local/[paymentId]` outcome controls. Barlow is installed locally through `@fontsource/barlow`.

Verification completed on 2026-07-16: web ESLint, TypeScript typecheck, and Next.js production build all pass. The commands emit a non-failing engine warning because the agent environment runs Node 26 while the repository requests Node 22.

The web server itself is runnable, but a fresh full stack is not yet operational from repository commands alone: `compose.yaml` starts PostgreSQL, Redis, and MinIO only; the API does not run migrations automatically; and there is no seed command or SQL fixture creating staff credentials, nurses, catalogue data, or demo records. Existing pre-populated databases may work, but a fresh database cannot support login or the demo journeys.

**How to apply:** Run the web app with Node 22 and `npx --yes pnpm@11.13.0 --filter @hospital/web dev` after setting `NEXT_PUBLIC_API_URL` and `NEXT_PUBLIC_WS_URL`. Before calling the full application demo-ready, provide and run database migration and deterministic seed commands, then start the API and worker alongside the infrastructure services.
