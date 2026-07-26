---
type: fact
tags: [prisma, sqlite, database]
created: 2026-07-09
agent: main
---

This project pins Prisma 7.8.0, which changed defaults in ways that break Prisma 5/6-era assumptions (most training data/tutorials):

- `prisma init` now defaults to the `prisma-client` generator, not `prisma-client-js`. It outputs to a custom path (here: `src/generated/prisma`) instead of `node_modules/@prisma/client`.
- `PrismaClient` now **requires an explicit driver adapter** — plain `new PrismaClient()` throws `PrismaClientInitializationError`. For SQLite, installed `@prisma/adapter-better-sqlite3` and construct with:
  ```ts
  import { PrismaBetterSqlite3 } from "@prisma/adapter-better-sqlite3";
  const adapter = new PrismaBetterSqlite3({ url: process.env.DATABASE_URL! });
  new PrismaClient({ adapter });
  ```
  The adapter strips a `file:` prefix from the URL itself, so the same `DATABASE_URL="file:./dev.db"` string works for both the Prisma CLI and the adapter.
- Seed config moved out of `package.json`'s `"prisma"` key and into `prisma.config.ts` (`migrations.seed: "tsx prisma/seed.ts"`). `prisma db seed` reads it from there now.
- `prisma init --datasource-provider sqlite` generates both `prisma.config.ts` and `.env`, and needs the `dotenv` package installed (it auto-adds `import "dotenv/config"` to the config file).

**Why:** ran into every one of these cold — `PrismaClientInitializationError` on first seed run, then had to hunt down the right adapter package name and API shape.

**How to apply:** in this repo, `src/lib/prisma.ts` and `prisma/seed.ts` are the reference implementations for the adapter pattern. Any new Prisma-backed project on this machine should check the installed Prisma major version before assuming the old `new PrismaClient()` no-args constructor works.
