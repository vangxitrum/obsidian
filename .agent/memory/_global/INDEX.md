# _global — memory index

Cross-project facts, preferences, workflow rules.

- [[hermes-obsidian-writes]] — route Obsidian vault writes (plans/specs/docs) through the hermes subagent; auto-write finalized plans to projects/<project>/plans/.
- [[hermes-kiro-credentials-dead]] — the pinned `kr/claude-sonnet-4.5-agentic` route 404s ("No active credentials for provider: kiro") as of 2026-08-28; `cx/gpt-5.6-sol` works, and how to list tool-capable routes.
- [[opencode-agent-notifications]] — OpenCode uses the shared dunst/tmux notification and pane-jump infrastructure through a global plugin.
- [[hermes-9router-provider]] — Hermes uses a named custom provider to discover and run 9router models.
- [[nvm-lazy-load-path-fix]] — nvm lazy-load via function shims broke Claude Code hooks (`node: not found`); fixed by prepending default node bin dir to PATH unconditionally in ~/.profile.
- [[go-edit-formatter-hook]] — the global PostToolUse hook re-wraps whole Go files; use Bash edits when a diff must stay minimal.
- [[docker-storage-layout]] — Docker 29.7.2 uses the containerd snapshotter, so image layers live in /var/lib/containerd; / is 78% full, /home is the spill target; migration script at ~/.local/bin/move-docker-storage.sh.
- [[m5-mcp-quirks]] — M5 MCP is plain HTTP+bearer (callable from cron); no data-read tool, so scripts return values by throwing into diagnostics; `math.roundToTick()` crashes the runtime.
- [[go-ldflags-x-struct-field-noop]] — `-X` on a struct field is silently dropped by the linker; ldflags targets must be plain string vars.
- [[go-private-modules-in-docker]] — building a Go image against gitlab.internal needs GOINSECURE (Go's own go-get probe), GOPRIVATE, and a key actually loaded into ssh-agent.
- [[nextjs-e2e-isolated-copy]] — e2e a Next.js app in a hardlinked repo copy (Turbopack panics on a symlinked node_modules); also how to build without clobbering a running dev server.
- [[renaming-a-live-repo-directory]] — moving a repo dir silently breaks running Next servers (absolute path cached at boot) and cron; checklist + the self-locating fix.
- [[anydesk-black-screen-two-causes]] — AnyDesk "waiting for image": xrandr `primary` on a disconnected output blocked capture, and a 1480 PMTU black hole (iface at 1500) killed video while audio/clipboard still worked.
- [[headless-chrome-no-localhost]] — headless Chrome from the agent shell cannot reach a localhost server (curl can); browser e2e unavailable, use curl + bundle greps instead.
- [[rabbitmq4-transient-queues]] — RabbitMQ 4 kills the connection on a transient non-exclusive queue declaration, which loops any client that re-declares on reconnect.
- [[testcontainers-fixed-host-port]] — a container stop/start test needs an explicitly bound host port, and the RabbitMQ image needs a non-guest user.
- [[cobra-print-goes-to-stderr]] - cobra `cmd.Print*` writes to stderr unless SetOut is set, so piped CLI output is empty; use `fmt.Fprint(cmd.OutOrStdout())`.
- [[testcontainers-docker-host-silent-fallback]] - a DOCKER_HOST that fails `docker info` is silently replaced by /var/run/docker.sock (cached per process); with dind + HOST_OVERRIDE every wait strategy times out on a refused port.
