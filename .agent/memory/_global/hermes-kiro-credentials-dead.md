---
type: fact
tags: [hermes, 9router, obsidian, tooling]
created: 2026-08-28
agent: main
---

Still dead on 2026-09-07 (third re-confirm: empty assistant reply, 0 tool calls, 0 files
written; `cx/gpt-5.6-sol` did all three parser-service v0.1.0 vault writes correctly).

Still dead on 2026-09-04 (re-confirmed: same HTTP 404, exit code 0, zero files written;
`cx/gpt-5.6-sol` completed the same vault write in 1m53s over 21 tool calls).

As of 2026-08-28 the `kr/claude-sonnet-4.5-agentic` model that [[hermes-obsidian-writes]]
pins for Obsidian vault writes **fails**:

```
HTTP 404: No active credentials for provider: kiro
Endpoint: http://127.0.0.1:20128/v1
```

The 9router itself is up - `curl -s http://127.0.0.1:20128/v1/models` answers fine.
The problem is only that the `kr/` (kiro) provider has no active credentials. At that
moment **every** tool-capable model on the router was `cx/*` (14 of them); there were
no claude/anthropic/sonnet routes at all.

**Working substitute:** `hermes chat -m cx/gpt-5.6-sol -q "..."` completed a
read-file → write-file → edit-INDEX vault task correctly (verified byte-identical
against the source). Still no `--yolo` needed.

Quick check before assuming a model exists:

```
curl -s http://127.0.0.1:20128/v1/models | python3 -c "
import sys,json
d=json.load(sys.stdin)
print([m['id'] for m in d['data'] if m.get('capabilities',{}).get('tools')])"
```

If kiro is re-authenticated the pinned model should work again; otherwise the
CLAUDE.md pin needs updating. Related: [[hermes-9router-provider]].

2026-09-11 recheck: `kr/claude-sonnet-4.5-agentic` still fails silently (the session ends after 1 message with 0 tool calls, and nothing is written). `cx/gpt-5.6-sol` wrote the aioz-stream plan revision successfully. After every hermes write, check that the file actually changed.
