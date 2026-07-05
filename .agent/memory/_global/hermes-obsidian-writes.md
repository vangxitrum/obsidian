---
type: preference
tags: [workflow, hermes, obsidian, plans, delegation]
created: 2026-07-05
agent: main
---

**Route Obsidian vault writes through the `hermes` subagent.**

**Why:** User wants file writes into the Obsidian vault (`/home/tuan/personal/tuan/`)
delegated to hermes (cost-sensitive backend), while the main model decides content
and destination.

**How to apply:**
- Any write of plans / specs / docs / notes into the vault → dispatch `hermes` via
  Task tool (`subagent_type: hermes`) to do the actual write. Main thread supplies
  exact target path + full content.
- **Automatically after every finalized plan** (planning/brainstorming done, plan
  approved): hermes writes it to `projects/<project>/plans/<YYYY-MM-DD>-<slug>.md`
  and refreshes `projects/<project>/INDEX.md`. No need to be asked.
- `<project>` = basename of work repo. Mapping: `work/depin-workspace/depin` →
  `projects/depin`; `work/backend` (aioz-map / "MAP") → `projects/backend`.
- **Exception:** `.agent/memory/` notes are written directly by the acting agent,
  NOT via hermes.

Also recorded in `~/.claude/AGENTS.md` (global CLAUDE.md). See project memory:
[[map-overview]], [[hub-overview]].
