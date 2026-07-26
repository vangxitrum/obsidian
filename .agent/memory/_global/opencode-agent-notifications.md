---
type: reference
tags: [opencode, notifications, tmux, dunst]
created: 2026-07-15
agent: main
---

OpenCode desktop/tmux notifications are provided by `/home/tuan/.config/opencode/plugins/agent-notify.ts`, which delegates to `/home/tuan/work/config/scripts/opencode-notify.sh` and the shared `agent-notify.sh`/`agent-state.sh` infrastructure.

The plugin tracks the active top-level session, marks its tmux window working on `chat.message`, sends input notifications through the native permission and question hooks, and sends turn-complete on `session.idle`. Existing tmux jump and jump-back bindings work because the shared notification script writes the same pane-aware jump state.

OpenCode must be restarted after plugin changes because plugins load at startup.
