---
type: fact
tags: [nextjs, turbopack, testing, e2e, tooling]
created: 2026-09-07
agent: main
---

To e2e-test a Next.js app without disturbing a dev server the user already has running, make an
isolated copy of the repo - but **hardlink `node_modules`, never symlink it**.

Turbopack panics on a symlinked `node_modules` that resolves outside the project directory:

```
FATAL: An unexpected Turbopack error occurred.
Symlink [project]/node_modules is invalid, it points out of the filesystem root
```

Hardlinks work and are instant, but only within one filesystem - so put the copy beside the
original (e.g. `/home/tuan/personal/.myapp-e2e`), not in `/tmp`, which is a different mount:

```bash
W=/home/tuan/personal/.myapp-e2e
rsync -a --exclude node_modules --exclude .next --exclude logs /path/to/repo/ "$W"/
cp -al /path/to/repo/node_modules "$W"/node_modules
cd "$W" && DATABASE_URL=... npx next dev -p 3099
```

Two other reasons this beats testing in place:
- `next dev` refuses to start a second server for the same directory ("Another next dev server
  is already running"), so a copy is the only way to run one with different env vars.
- A copied SQLite file means an e2e that writes rows cannot pollute the real database.

The same copy trick lets `next build` run without clobbering the `.next` a long-running dev
server is using. Delete the copy afterwards; it is ~800MB of hardlinks.

Seen while building [[live-order-sidecar]].
