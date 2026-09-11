---
type: fact
tags: [tooling, hooks, go, claude-code]
created: 2026-08-17
agent: main
---

The global `PostToolUse` hook in `~/.claude/settings.json` runs nvim + conform on every `.go`
file touched by the Write/Edit/MultiEdit tools. Conform is configured with a line-rewrapping
formatter (golines-style), so a one-line Edit to a Go file comes back with the whole
surrounding function re-wrapped - tens of unrelated diff lines.

**How to apply:** when a Go change must stay minimal (lint burndown, a one-line fix, anything
that will be code-reviewed), make the edit through Bash (`perl -i`, `python3`, heredoc)
instead of the Edit tool. The hook matches on the tool name, not the file, so shell edits are
untouched. The tool-based path is fine when the file is being rewritten anyway.

Seen while porting the lint CI in [[lint-ci-golangci]].

**Also (2026-09-10):** the hook runs goimports after every Edit/Write. An import that isn't used *yet* is dropped, and a stale one may be re-added. So add an import in the same edit as its first use, or rewrite the import block after all the usages exist (e.g. with a python exact-replace).
