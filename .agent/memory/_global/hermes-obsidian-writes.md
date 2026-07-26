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
- Use the **hermes CLI**, NOT the `hermes` Task subagent. The subagent is forced onto
  a native Anthropic tier by the harness (runs Haiku, never a Hermes/router model) —
  confirmed broken for this purpose.
- Working command (verified 2026-07-06):
  `hermes chat -m kr/claude-sonnet-4.5-agentic -q "<instruction>"`
  Wrote to the vault with NO `--yolo` and no wrapper/allow-rule needed.
- **Model matters** (backend = 9router at `127.0.0.1:20128`, `kr/` prefixed ids):
  - default `nvidia/moonshotai/kimi-k2.6` → empty after tool calls (too weak). Avoid.
  - `nvidia/deepseek-ai/deepseek-v4-pro` → 404 no creds.
  - `kr/claude-sonnet-4.5-agentic` → works (agentic, drives tools + skills). Use this.
- hermes has the `personal-obsidian` skill; instruct it e.g. "using personal-obsidian
  skill, append '<text>' to <file> with today's date". It resolves vault paths itself.
- For plans/specs, pass exact target path + content: dispatch
  `hermes chat -m kr/claude-sonnet-4.5-agentic -q "write EXACTLY <content> to <path>"`.
- **Automatically after every finalized plan** (planning/brainstorming done, plan
  approved): hermes writes it to `Projects/<project>/plans/<YYYY-MM-DD>-<slug>.md`
  and refreshes `Projects/<project>/INDEX.md`. No need to be asked.
  - **Capital-P `Projects/`** — the PARA top-level dir. NEVER lowercase `projects/`
    (a legacy dir being deleted; writing there strands the file in the wrong place).
- `<project>` = basename of work repo. Mapping: `work/depin-workspace/depin` →
  `Projects/depin`; `work/backend` (aioz-map / "MAP") → `Projects/backend`.
- **Updating an existing INDEX.md via hermes:** do NOT ask hermes to read-and-merge
  (its read can miss the file and clobber it to a fresh stub — happened 2026-07-14,
  wiped a vault INDEX). Instead stage the full corrected file content in a scratchpad
  file yourself, then tell hermes to write it VERBATIM/byte-identical to the target.
- **Exception:** `.agent/memory/` notes are written directly by the acting agent,
  NOT via hermes.

Also recorded in `~/.claude/AGENTS.md` (global CLAUDE.md). See project memory:
[[map-overview]], [[hub-overview]].
