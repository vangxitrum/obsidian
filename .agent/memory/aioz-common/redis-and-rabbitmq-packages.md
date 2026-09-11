# redis and rabbitmq packages (added 2026-09-09)

Two client packages, deliberately asymmetric.

`redis` is thin: go-redis already owns reconnection, pooling and command retry,
so the package adds only construction, a startup ping (`Open` dials and pings),
timeouts, a monkit hook that counts `redis.Nil` as a miss rather than an error,
and `Close`. `Client` embeds `*goredis.Client`; no command is wrapped and there
is no `Cache` interface. No cluster support on purpose.

`rabbitmq` goes up to the message: `Conn` (supervised, exponential backoff,
reconnects forever), `Topology` (re-declared on every reconnect, DLX/DLQ
injection), `Publisher` (per-publish deferred confirms, so concurrent
publishers cannot steal each other's acks), `Consumer` (bounded pool,
`Requeue(err)` vs bare error ack policy, drain at shutdown). It replaces eight
hand-rolled wrappers across the estate, three of them stale forks.

Two design points that are easy to get wrong again:

- `Conn.Session(ctx, fn)`: `fn` returning **nil** ends the session for good.
  A `Run` body that blocks on `<-ctx.Done()` must return `ctx.Err()`, or the
  publisher/consumer silently stops after the first reconnect.
- A mandatory publish waits a 25ms grace after its confirm for a possible
  `basic.return`. The broker sends the return before the ack, but amqp091
  delivers returns on a separate path, so a strictly non-blocking check
  sometimes reports an unroutable message as sent.

Integration tests use testcontainers with a docker probe that skips unless
`AIOZ_REQUIRE_DOCKER=1`; CI runs them in a dind stage.

Related: [[rabbitmq4-transient-queues]], [[testcontainers-fixed-host-port]]
