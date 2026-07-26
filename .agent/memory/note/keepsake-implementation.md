---
type: fact
tags: [architecture, pwa, local-first]
created: 2026-07-19
agent: main
---

Keepsake in `/home/tuan/personal/note` is a React 19 and TypeScript mobile-first PWA built with Vite, TipTap, vite-plugin-pwa, Fastify, Postgres, and Playwright.
It uses built-in email/password accounts and stores notes, folders, tags, tasks, reminders, links, and preferences durably in per-user Postgres data.
The development server normally runs at `http://localhost:5173`.

**How to apply:** Source `/home/tuan/.nvm/nvm.sh` before npm commands because npm is not always present on the default shell PATH.

The LAN URL `http://10.0.0.67:5173` is an insecure origin where `crypto.randomUUID` and service workers are unavailable.
Keepsake uses `src/id.ts` to generate IDs safely in that environment, so note, task, tag, and folder creation must not call `crypto.randomUUID` directly.

IndexedDB schema version 3 adds customizable tag colors and nested tag relationships through `TagRecord.parentId`.
Tag movement must use `moveTag` in `src/data.ts` so parent-child cycles remain impossible.

Vite development and preview servers set `allowedHosts: true` so user-managed tunnel hostnames are accepted.
No tunnel process is managed by the project itself.

Port `5173` runs the Dockerized Fastify application and Postgres API while the user is actively using or tunneling Keepsake.
The live deployment is managed with `docker compose up -d --build app`, and the tunnel points to that application port.

IndexedDB schema version 4 adds up to three active `GoalRecord` items, task-to-goal links, daily or selected-weekday recurrence, and `lastCompletedAt`.
Completing a recurring task advances the same task to its next scheduled occurrence instead of creating a copy.

IndexedDB schema version 5 adds `NoteRecord.parentTaskId` for nested child tasks.
Task movement must use `moveTask` in `src/data.ts` to prevent parent-child cycles.
Tag colors use a named preset palette plus a custom color input in both creation and row editing.
The task editor can create a nested subtask directly and carries the parent task's goal into it.
Selecting a nested tag stores its full ancestor chain; unchecking an ancestor also clears its selected descendants.
Archiving or restoring a parent task applies to its complete descendant subtree in one IndexedDB transaction.
Task editor changes are held in memory and written transactionally only when Done is clicked; Escape, Back, and backdrop dismissal discard edits, and dismissed new tasks are deleted.
Done persists the task record before maintaining its optional reminder, shows Saving/error feedback, and closes the editor immediately before asynchronous history cleanup.
Existing goals expose an edit-details action for changing their title, description, and color.
Each active goal links to a dedicated detail page showing full task content, status, due date, recurrence, hierarchy, progress, and controls for completion, editing, and subtask creation.
Existing controlled PWA pages reload automatically when a newly activated service worker takes control, and service-worker updates are checked hourly.
Postgres is the sole durable store; the client keeps only an in-memory server snapshot and refreshes it when the page becomes visible and every 60 seconds.
Account passwords use scrypt hashes, sessions use hashed opaque HTTP-only cookies, and registration/login endpoints are rate limited.
Docker Postgres uses the `keepsake-postgres` volume and a standard `DATABASE_URL`, allowing a later switch to Supabase Postgres without frontend changes.
The first authenticated load can import legacy IndexedDB data after explicit confirmation, then removes the local database only after successful import.
