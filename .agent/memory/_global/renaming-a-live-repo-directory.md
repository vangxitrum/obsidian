---
type: fact
tags: [ops, nextjs, cron, refactoring, gotcha]
created: 2026-09-08
agent: main
---

Renaming a repo directory does **not** move the things already pointing at the old path. It
looks fine for hours, then everything fails at once.

Hit on 2026-09-07 renaming `/home/tuan/personal/trace` -> `/home/tuan/personal/binance`
(see [[live-order-sidecar]]). `mv` preserves the inode, so each running process's `cwd` followed
the rename correctly - which is exactly why it seemed safe. But:

- **Next.js resolves an absolute project path at boot and caches it.** The dev server (3015) then
  threw `ENOENT: no such file or directory, scandir '/home/tuan/personal/trace/app'` on every
  file scan, and the production server (3011) threw `ChunkLoadError` /
  `MODULE_NOT_FOUND` for `.next/server/chunks/...` under the old path. `readlink /proc/<pid>/cwd`
  showed the NEW path, so cwd is a red herring - check `ps -o cmd=` on the **parent**, which
  still carries the old absolute path.
- **The orphaned dev server recreated the old directory** to write `.next` into. A directory
  reappearing after you deleted it is the tell; compare inodes (`stat -c '%i %n'`) to prove it is
  a new one, not a failed rename.
- **crontab keeps the old absolute path** and simply stops running. Silent: no error anywhere,
  the job just never fires. 15 hours of captures were lost before anyone noticed.

## Checklist when moving a repo
1. `pgrep -a -f <appname>` and stop every server started from the old path BEFORE the move.
2. `crontab -l | sed 's|/old/path|/new/path|g' | crontab -` (back up `crontab -l` first).
3. `grep -rn "/old/path" .` across the repo - hardcoded paths hide in shell wrappers and in
   `child_process` calls (`cwd:` options especially).
4. Restart servers from the new path and confirm with `ps -o cmd=` on the parent process.
5. Verify cron actually fires: watch the log file's mtime change, do not assume.

## The durable fix
Never write the repo root out. In shell wrappers:

```bash
cd "$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
```

In Node ESM:

```ts
join(dirname(fileURLToPath(import.meta.url)), "..")
```

Then only the crontab entry itself needs the absolute path, because cron requires one.
