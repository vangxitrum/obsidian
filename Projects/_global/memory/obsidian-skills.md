---
type: reference
tags: [obsidian, skills, memory]
created: 2026-07-05
agent: main
---

# Obsidian skills & vault layout

Vault root: `/home/tuan/personal/tuan/`.

Two skills write here:
- **obsidian-inbox** — quick capture. `Inbox/<slug>.md` (one file per note, with frontmatter).
- **obsidian-daily-task-log** — daily notes. `Daily/YYYY-MM-DD.md` (tasks under `## Tasks` as `- [ ]`, log entries under `## Log`).
- **obsidian-memory** — agent shared memory, per project: `projects/_global/memory/` (cross-project) + `projects/<basename $PWD>/memory/` (project-specific), each with `INDEX.md`.

Restructured 2026-07-05 from flat `agent-memory/` to `projects/*/memory/`. `AGENTS.md`
points every agent at the new paths.
