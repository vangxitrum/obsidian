---
type: decision
tags: [shell, nvm, node, zsh, claude-code, hooks]
created: 2026-07-23
agent: main
---

`~/.profile` (sourced by `~/.zshrc`) previously lazy-loaded nvm by defining `node`/`npm`/`npx`/`nvm` as shell functions that sourced `nvm.sh` only on first call. This broke any non-interactive subprocess that needs `node` on `PATH` — notably Claude Code hooks, which run via a bare `/bin/sh -c "..."` and never trigger a zsh function. The `caveman` plugin's `SessionStart`/`UserPromptSubmit` hooks (`node "${CLAUDE_PLUGIN_ROOT}/src/hooks/*.js"`) failed with `/bin/sh: 1: node: not found`.

**Why:** shell functions are local to the process that defines them and are never inherited by child processes. `PATH` is the only thing that propagates. Lazy-loading via function shims trades correctness for speed in a way that only works for interactively-typed commands, not for anything Claude Code, cron, or other tools spawn as subprocesses.

**Fix applied:** in `~/.profile`, resolve nvm's default-version `bin/` dir cheaply (read `$NVM_DIR/alias/default`, match against `$NVM_DIR/versions/node/*`, no `nvm.sh` sourcing) and prepend it to `PATH` unconditionally at shell start. `node`/`npm`/`npx` (and any other node-based global CLI installed there, e.g. `ccr`, `9router`, `gemini`) now resolve via plain PATH lookup for every process. Only `nvm` itself (not a real binary) stays a lazy function, since it's only ever invoked interactively. Verified: `zsh -i -c 'exit'` stays ~0.5s (no eager `nvm.sh` sourcing reintroduced), and `sh -c 'node --version'` succeeds post-fix where it previously failed.

**How to apply:** if node-based Claude Code hooks/plugins (or any non-interactively-spawned node tool) mysteriously report "node: not found" again, check whether `~/.profile`'s PATH-prepend block for nvm's default bin dir is still intact — don't reach for function-shim lazy-loading for `node`/`npm`/`npx` themselves, only for `nvm` the version-switcher.
