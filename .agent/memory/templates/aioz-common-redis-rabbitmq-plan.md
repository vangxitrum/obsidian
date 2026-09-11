# aioz-common redis + rabbitmq client packages (plan, 2026-09-09)

Plan note: `Projects/templates/plans/2026-09-09-aioz-common-redis-rabbitmq-clients.md`.
Not yet implemented as of 2026-09-09.

## Vault mapping (non-obvious)

The repo is `/home/tuan/work/templates/aioz-common` but its vault folder is
`Projects/templates/`, **not** `Projects/aioz-common/` (which does not exist).
Same for agent memory: `.agent/memory/templates/`. The mapping is by the parent
dir `templates`, not by the repo basename.

## Decisions the user made

- Thin wrapper + lifecycle only. No Cache/Queue interface, no JSON helpers, no
  distributed lock, no publisher/consumer framework.
- `redis/go-redis/v9` + `rabbitmq/amqp091-go`. Confirmed as what every current
  first-party repo already uses; nobody uses `wagslane/go-rabbitmq`.
- testcontainers-go, with a **new dind stage** in `.gitlab-ci.yml` (the existing
  test job has no docker and no services). Runner must be privileged.
- Redis config: DSN plus discrete overrides, DSN wins.

## Facts worth keeping

- Five separate RabbitMQ wrappers exist across AIOZ repos. The hub one
  (`depin-workspace/hub/.../rabbitmq/connection.go:174-218`) has a real bug:
  `notifyClose` is reassigned under a lock the watcher does not hold, so after
  the first reconnect it watches a dead channel and never reconnects again.
  `aioz-map/parser-service/internal/queue/consumer.go:55-74` is the clean model:
  reconnect as the outer loop, one connection per attempt.
- `amqp091` has **no `DialContext`**. Dial timeout must go through
  `amqp.Config.Dial`.
- `NotifyClose`'s channel must be **buffered**: amqp091 blocks sending the close
  reason during shutdown, so an unbuffered one deadlocks the connection.
- RabbitMQ's `guest` user is **loopback-only**. From another container it fails
  with `ACCESS_REFUSED`, which reads like a wrong password. Set
  `RABBITMQ_DEFAULT_USER`/`PASS` in any container-based test.
- A go-redis health method must be named `Health`, not `Ping`: `Ping` would
  shadow the embedded `(*redis.Client).Ping` with a different return type.
- `redis.Nil` is a cache miss. Any metrics hook must exclude it before counting
  errors.
- Do not chain a `monkit.StatSource` from a constructor: `Scope.Chain` appends
  unconditionally, so every pool a test opens double-counts. Same trap
  `stats/monitor.Register` and `version.Register` close. See
  [[aioz-common-version-build-stamp]].
- A DSN field embeds the password, so it must be `secret:"optional"` or
  `SaveConfig` writes the password into `config.yaml`. See
  [[aioz-common-config-secret-guard]].
- testcontainers is acceptable as a dependency despite ~50 modules: module-graph
  pruning keeps a test-only dep of an imported package out of the consumer's
  build list entirely. Different from the `base58` vendoring case, which was a
  build dep. See [[aioz-common-consolidation]].
- Avoid `//go:build integration`: golangci-lint does not see tagged files, so
  they rot. Use a docker probe that skips, plus `AIOZ_REQUIRE_DOCKER=1` in CI to
  turn the skip into a failure.
