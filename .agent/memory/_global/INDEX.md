# _global — memory index

Cross-project facts, preferences, workflow rules.

- [[hermes-obsidian-writes]] — route Obsidian vault writes (plans/specs/docs) through the hermes subagent; auto-write finalized plans to projects/<project>/plans/.
- [[opencode-agent-notifications]] — OpenCode uses the shared dunst/tmux notification and pane-jump infrastructure through a global plugin.
- [[hermes-9router-provider]] — Hermes uses a named custom provider to discover and run 9router models.
- [[nvm-lazy-load-path-fix]] — nvm lazy-load via function shims broke Claude Code hooks (`node: not found`); fixed by prepending default node bin dir to PATH unconditionally in ~/.profile.
