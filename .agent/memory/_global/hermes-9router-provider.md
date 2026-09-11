---
type: fact
tags: [hermes, 9router, models]
created: 2026-07-22
agent: main
---

Hermes connects to the local 9router OpenAI-compatible endpoint through the named `custom:9router` provider in `~/.hermes/config.yaml`. A bare `model.provider: custom` entry only exposed the current model in Hermes' picker; `custom_providers` with live discovery exposes the router catalog. The default verified model is `cx/gpt-5.6-sol`. Keep the credential in `~/.hermes/.env` and do not copy it into memory notes.

**2026-08-25:** `kr/claude-sonnet-4.5-agentic` - the model CLAUDE.md designates for Obsidian vault writes - fails with `HTTP 404: No active credentials for provider: kiro` at `http://127.0.0.1:20128/v1`. The kiro upstream credential behind the `kr/` prefix has expired or been revoked; 9router itself is up. Until it is refreshed, delegating vault writes to the hermes CLI fails fast (non-retryable 404, ~3s), so write vault files directly instead. Re-check with a trivial `hermes chat -m kr/claude-sonnet-4.5-agentic -q hi` before relying on it again. See [[hermes-obsidian-writes]].
