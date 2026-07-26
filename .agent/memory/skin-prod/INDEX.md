# skin-prod — memory index

Simple Next.js skincare quiz + product recommendation app.

- [[project-overview]] — what the app is, stack, architecture decisions, verification status
- [[base-ui-shadcn-variant]] — shadcn/ui here is Base UI, not Radix; asChild/nativeButton/RadioGroup gotchas
- [[prisma-7-driver-adapters]] — Prisma 7 requires explicit driver adapters (SQLite via @prisma/adapter-better-sqlite3), seed config moved to prisma.config.ts
- [[nextjs-lan-dev-origin-block]] — Next 15+ silently blocks dev JS when accessed via LAN IP instead of localhost; needs allowedDevOrigins
