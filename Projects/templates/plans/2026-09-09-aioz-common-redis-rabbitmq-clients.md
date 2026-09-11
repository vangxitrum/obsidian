# aioz-common: add `redis` and `rabbitmq` client packages

## Context

The point of these two packages is to be **the** client every AIOZ service uses
to talk to Redis and RabbitMQ, connection handling and error handling included,
so that no service writes that code again. The measure of success is how much
existing code they let a repo delete, not how small the new packages are.

An inventory of `/home/tuan/work` establishes what is actually being rewritten.

**RabbitMQ: 8 hand-rolled wrappers, ~3,000 LOC**, three of them stale forks of
each other:

| Repo | Wrapper | Lib | LOC |
|---|---|---|---|
| depin-workspace/hub (worker manager) | `internal/worker/manager/infrastructure/rabbitmq/{connection,publisher,broker}.go` | amqp091 v1.10 | 444 |
| depin-workspace/hub (eventbus) | `pkg/eventbus/integration_bus/` + a verbatim copy under `internal/storage/infrastructure/eventbus/` | watermill-amqp | 320 ×2 |
| aioz-map/crawler-service | `internal/utils/queue/{rabbitmq,worker,consumer,config,model}.go` | amqp091 v1.12 | 736 |
| map/crawler | `internal/pkg/queue/` — stale fork of crawler-service | amqp091 v1.12 | 453 |
| aioz-map/parser-service | `internal/queue/{consumer,dlq}.go` + `internal/worker/dispatcher.go` | amqp091 v1.12 | 396 |
| stream/aioz-stream | `rabbitmq/main.go` | **streadway** v1.1 | 352 |
| stream/aioz-stream-core | `internal/utils/rabbitmq/rabbitmq.go` — stale fork of aioz-stream | streadway v1.1 | 364 |
| go-aioz-media | inline in `censor/controller.go:99-260` (byte-identical copy in depin-workspace) | amqp091 v1.8 | ~130 |

Seven independent reconnect strategies, **every one a fixed delay with no
backoff or jitter**, and three live bugs that a shared implementation removes by
construction:

- **hub steals publisher confirms.** One unbuffered `notifyConfirm`
  (`connection.go:122`) is read by any publisher (`publisher.go:92-104`) with no
  correlation to a `DeliveryTag`, so concurrent publishes consume each other's
  acks and a failed publish can report success.
- **hub self-deadlocks on the error path.** `Close()` (`connection.go:129`) takes
  `c.mutex` and is called from inside `Connect()` (`connection.go:63`), which
  already holds it.
- **hub never reconnects twice.** `StartReconnectLoop` (`connection.go:174-218`)
  reassigns `notifyClose` under a lock the watching goroutine does not hold, so
  after the first reconnect it selects on a dead channel forever. Its retry is a
  fixed `time.Sleep` with a finite attempt cap that then gives up silently.
- **aioz-stream races and loses messages.** `Reconnect` (`main.go:310-312`)
  replaces `conn`/`channel` with no mutex while `Publish`/`Consume` read them;
  it polls `IsClosed()` every 5s instead of using `NotifyClose`; it publishes
  **transient** (no `DeliveryMode`) and consumes with **autoAck** and no drain.

**Redis: no wrapper anywhere, and nothing to de-duplicate.** Four repos use
`go-redis/v9` (`go-aioz-media`, `depin-workspace/hub`, `stream/aioz-stream`,
`ipfs/pinning-service`), and two of those have no live production path at all —
`NewRedisMetrics`/`NewRedisApiCountStore` have zero callers, and hub's
`NewUsageTracker` is called only from tests. Across every repo exactly three
options are ever set — `Addr`, `Password`, `DB`. **No repo sets a dial, read or
write timeout, a pool size, `MaxRetries`, or TLS. No repo calls `Close()`. No
repo has a single Redis metric or a Redis-aware readiness probe.** One repo pings
at startup, via `panic` (`stream/aioz-stream/internal/config/redis.go:23-25`).
Passwords travel as plain config strings
(`depin-workspace/hub/internal/storage/config/config.go:16` is literally
`Redis struct{ Conn string }`) or are hardcoded empty
(`ipfs/pinning-service/main.go:141`).

So the two packages are deliberately **asymmetric**, and that asymmetry is the
main decision in this plan:

- **`redis` stays thin.** go-redis already owns reconnection, pooling and command
  retry. What is missing is construction, a startup ping, timeouts, metrics and
  `Close` — nothing above the client. Every advanced use in the estate (hub's
  `WATCH`+retry, `SCAN` sweeps, `redis:"..."` struct-tag `HGETALL.Scan`,
  `TxPipelined`, `MGET`, `INCRBY`) is direct go-redis that embedding already
  supports. There is **no** Redis pub/sub, streams, Lua, Sentinel, Cluster,
  `redsync` or `AddHook` in first-party code.
- **`rabbitmq` goes up to the message.** Connection supervision alone would let
  each repo delete only its ~70-line dial block and keep 150-250 lines of
  publisher and consumer. So the package owns **Conn + Topology + Publisher +
  Consumer**, which is exactly the set of features that appears in two or more
  repos.

Decisions taken before planning:

- **`github.com/redis/go-redis/v9` and `github.com/rabbitmq/amqp091-go`**, what
  every current first-party repo already uses. aioz-stream's `streadway/amqp` is
  abandoned and migrates.
- **testcontainers-go integration tests**, run in CI by a new dind job.
- **Redis config: DSN plus discrete overrides.**
- **Library only.** Migrating the 8 wrappers is separate follow-up work per repo.
  This plan ships the packages, their tests and their docs.

### Scope boundary for `rabbitmq`

In scope, because two or more repos wrote it:

| Feature | Written independently in |
|---|---|
| Dial + reconnect loop | hub, crawler-service ×2, map/crawler, parser-service, aioz-stream, go-aioz-media |
| Connection `NotifyClose` watcher | hub, crawler-service, parser-service |
| JSON publish, `ContentType`, `DeliveryMode: Persistent` | hub, crawler-service, parser-service, aioz-stream, go-aioz-media |
| Publisher confirms + wait-with-context | hub, crawler-service |
| Publish retry | crawler-service, aioz-stream, go-aioz-media (three different strategies) |
| Qos/prefetch with a `<1 → 1` clamp | crawler-service, parser-service, hub, eventbus |
| Bounded worker pool (`sem` + `WaitGroup`) per delivery | crawler-service `consumer.go:91-125`, parser-service `dispatcher.go:64-98` — near line-for-line identical |
| Manual ack, error → Nack policy | crawler-service, parser-service, eventbus |
| DLX/DLQ topology (`x-dead-letter-exchange`/`-routing-key`) | crawler-service ×2, map/crawler, parser-service — three near-identical implementations |
| Topology re-declared on every reconnect | all seven |
| Graceful drain (cancel consume, wait for in-flight) | crawler-service, parser-service |
| Queue/exchange config struct | hub, crawler-service, map/crawler, parser-service |

Out of scope, because exactly one repo has it, and each is a service policy
rather than a transport concern. The README names them so a migrating repo knows
what it must keep:

- Disk spool and replay of failed publishes (aioz-stream `main.go:98,327`).
- App-level retry with attempt counting, transient/permanent classification and a
  `DLQEntry{ArrivedAt,FailedAt,Attempts,Error,EventBody}` written to a diagnostic
  DLQ over a per-message connection (parser-service `internal/worker/router.go`,
  `internal/queue/dlq.go`). The **broker** DLQ is in scope; this second
  application-level one is not.
- The `event_id`/legacy `event-id` correlation header contract between
  crawler-service (`worker.go:51-54`) and parser-service (`dispatcher.go:110`).
  `Message.Headers` and `Message.MessageID` carry it; the package does not
  define or require it. Codifying a house envelope is a separate decision.
- Watermill's `TopologyBuilder`, consumer-group queue naming, idempotency-Ack on
  `ErrAlreadyExists` (hub eventbus).
- Named multi-consumer fan-out with `Cancel(name)`, autoAck, non-durable queues
  (aioz-stream).
- Request/response queue pair returning a caller-invoked `ackFunc`
  (go-aioz-media `controller.go:206`).

---

## 1. `redis/`

Files: `redis/common.go` (`Error`, `mon`), `redis/redis.go` (package doc,
`Config`, `Client`, `Open`, `OpenWith`, `Health`, `Close`, `Stats`),
`redis/monkit.go` (the go-redis `Hook`), `redis/redis_test.go` (docker-free),
`redis/client_test.go` (integration), `redis/docker_test.go` (endpoint helper),
`redis/README.md`.

Import alias throughout: `goredis "github.com/redis/go-redis/v9"`, since our
package is also called `redis`. Callers who need `goredis.Nil` hit the same
collision, so the README says so.

### Package doc, the why

go-redis dials lazily. A service with a typo in its address, an expired password
or a firewall in the way starts perfectly cleanly and reports itself healthy; the
failure surfaces on the first business request, hours later, attributed to
whatever code happened to make it. `Open` dials and pings, so a misconfigured
cache fails at startup next to the configuration that caused it. The second thing
it prevents is a silently unobservable client: the pool carries a monkit hook, so
command latency and error rates exist for every service without anyone
remembering to add them, and the hook knows `redis.Nil` is a cache miss rather
than an error, which is the one thing a hand-rolled counter always gets wrong.

### Config

```go
type Config struct {
	Name string `help:"instance name; tags this client's metrics so two pools can be told apart" default:"redis"`

	URL      string `help:"redis connection URL (redis:// or rediss://); when set it overrides address, db, username and password" secret:"optional"`
	Address  string `help:"redis server address as host:port" default:"127.0.0.1:6379"`
	DB       int    `help:"logical database number" default:"0"`
	Username string `help:"ACL username; empty uses the default user" default:""`
	Password string `help:"redis password; empty means no AUTH" secret:"optional"`

	ClientName string `help:"name this connection reports to CLIENT LIST" default:""`

	PoolSize     int `help:"maximum socket connections; 0 uses 10 per CPU" default:"0"`
	MinIdleConns int `help:"connections kept open while idle" default:"0"`
	MaxRetries   int `help:"retries per command before giving up; -1 disables retrying" default:"3"`

	DialTimeout  time.Duration `help:"timeout establishing a connection" default:"5s"`
	ReadTimeout  time.Duration `help:"socket read timeout; -1 disables" default:"3s"`
	WriteTimeout time.Duration `help:"socket write timeout; -1 disables" default:"3s"`
	PoolTimeout  time.Duration `help:"how long a caller waits for a free connection; 0 uses ReadTimeout+1s" default:"0"`

	TLS           bool `help:"connect with TLS" default:"false"`
	TLSSkipVerify bool `help:"do not verify the server certificate; test use only" default:"false" hidden:"true"`

	RequireOnStart bool          `help:"refuse to start if the server cannot be reached" default:"true"`
	HealthTimeout  time.Duration `help:"bound on the startup ping and on Health" default:"3s"`

	Metrics bool `help:"record monkit series for commands and the pool" default:"true"`
}
```

**`URL` is `secret:"optional"`, not a plain string.** A DSN embeds the password,
so the whole field is a credential; tagging it otherwise would have
`config.SaveConfig` write the password into `config.yaml`. This is the one real
cost of supporting a DSN, and it is why the discrete fields stay: they are what
makes the generated config file self-documenting, which is the point of the
`config` package. Both DSN styles are in use today — `redis.ParseURL` in
go-aioz-media (`metrics/redis.go:28`) and hub's `redis.conn`, versus discrete
host/port/DB fields in aioz-stream and pinning-service — so both must work.

`Password` is `secret:"optional"`, not `"true"`. A local Redis with no password is
normal and `"true"` would make every service refuse to boot without one. Bare
`secret:"optional"` with no default tags is legal: `checkSecretTag`
(`config/secrets.go:39-66`) only rejects a non-empty `default`/`releaseDefault`,
and `getDefault` returns `""` when it finds no opposite tag. If a dev value is
ever wanted the sanctioned shape is
`secret:"optional" devDefault:"devpass" releaseDefault:""`; a lone `devDefault`
panics.

`DB` is not decoration. aioz-stream runs **two clients against the same host on
different logical DBs** (`internal/config/redis.go:17,27`), which under this
package is two `Open` calls with different `Name` and `DB` — and the distinct
`Name` is what keeps their metrics apart.

```go
// Options translates the Config into go-redis options. Exported so a service
// with an unusual requirement can start from the configured pool and adjust one
// field rather than abandoning the Config entirely.
func (config Config) Options() (*goredis.Options, error)
```

When `URL` is set it is parsed with `goredis.ParseURL`, and the pool, timeout and
retry fields are applied **on top of** the result, so a DSN never silently
reverts timeouts to go-redis defaults. Otherwise the discrete fields build the
options directly. Empty `Address` with an empty `URL` is
`Error.Errorf("address is empty")`.

### API

```go
// Client is a configured redis connection pool. The embedded *redis.Client is
// the API: this type adds construction, a health check and metrics, and
// deliberately wraps no commands.
type Client struct {
	*goredis.Client

	log    *slog.Logger
	config Config
	name   monkit.SeriesTag
}

func Open(ctx context.Context, log *slog.Logger, config Config) (*Client, error)
func OpenWith(ctx context.Context, log *slog.Logger, opts *goredis.Options, config Config) (*Client, error)
func (client *Client) Health(ctx context.Context) error
func (client *Client) Close() error
func (client *Client) Stats(cb func(key monkit.SeriesKey, field string, val float64))
```

- `*redis.Client` is **embedded**, so `client.Get(ctx, k)` works and the wrapper
  adds no method surface to keep in sync with upstream. Embedding is what makes
  hub's `Watch`, `TxPipelined`, `SScan` and `HGETALL`-into-struct keep working
  unchanged, which is the requirement that rules out a `Cache` interface.
- `OpenWith` is the escape hatch for a caller whose endpoint arrives as a DSN
  from a platform in a shape `ParseURL` does not cover; `config` then supplies
  only the health and metrics settings.
- The health check is called **`Health`, not `Ping`**, on purpose. A
  `Ping(ctx) error` would shadow the embedded `Ping(ctx) *StatusCmd` with a
  different return type, which is a trap for anyone reading a call site. The raw
  command stays reachable as `client.Client.Ping(ctx)`.
- `Open` pings before returning. On failure, with `RequireOnStart` it closes the
  pool and returns the error; without, it logs a warning and returns the client.
  It never logs `opts.Password`. This replaces aioz-stream's
  `panic("failed to connect redis")` with an error the composition root handles.
- `Stats` implements `monkit.StatSource` over `(*redis.Client).PoolStats()`:
  `hits`, `misses`, `timeouts`, `total_conns`, `idle_conns`, `stale_conns`, under
  `monkit.NewSeriesKey("redis_pool")` tagged by `Config.Name`, the same
  per-instance tagging as `lrucache/cache.go:153`. Follow `version/stats.go:65`
  for the shape.
- **`Open` does not chain `Stats` itself.** `Scope.Chain` appends
  unconditionally, so a constructor that registered would double-count every pool
  a test opened. The service writes
  `monkit.Default.ScopeNamed("redis").Chain(client)` once, which is the same trap
  `stats/monitor.Register` and `version.Register` exist to close.

**No Cluster, Sentinel or Ring support.** There is exactly one reference in
first-party code, commented out (`ipfs/pinning-service/main.go:137-139`), with a
`RedisConns []string` config field feeding it that was never enabled. Adding a
`UniversalClient` would change the embedded type from `*goredis.Client` to an
interface and cost every caller the concrete methods, to serve nobody. If it is
ever needed it arrives as a separate `OpenCluster` returning a distinct type.
Note for a future migration: hub's `SCAN`/`SSCAN` keyspace sweeps
(`charge.go:44,153`, `reserve.go:215,241,287`) do not behave the same against a
cluster, so this is not a swap that can be made silently.

### `redis/monkit.go`

```go
type hook struct {
	name     monkit.SeriesTag
	duration *monkit.DurationVal
	pipeline *monkit.DurationVal
	pipeLen  *monkit.IntVal
}

func (h *hook) DialHook(next goredis.DialHook) goredis.DialHook
func (h *hook) ProcessHook(next goredis.ProcessHook) goredis.ProcessHook
func (h *hook) ProcessPipelineHook(next goredis.ProcessPipelineHook) goredis.ProcessPipelineHook
```

Three details that are the whole reason this file exists:

- **`redis.Nil` is not an error.** `ProcessHook` checks
  `err != nil && !errors.Is(err, goredis.Nil)` before firing
  `mon.Event("redis_command_failed", ...)`. A cache miss counted as an error
  makes every dashboard useless.
- **The `cmd` tag rides on the error event only, never on the histogram.** The
  `*DurationVal` is created once per client, so the hot path is
  `h.duration.Observe(...)` with no scope lookup. Tagging the histogram by
  command name would put a locked map lookup in `Scope.DurationVal` on every
  single redis call, for a per-command latency breakdown nobody reads. The
  per-command *error* breakdown is the one you want, and errors are rare.
- **The hook uses `time.Now()`, not `time2.Now(ctx)`.** `time2` is the convention
  where a test needs to skip a wait; here it would add a `ctx.Value` lookup to
  every command to make a measurement testable that nothing tests. `time2` does
  earn its place in the rabbitmq backoff.

`mon.Task()` is deliberately not used in the hook: it would open a monkit span
per redis command and attribute it to the caller's context.

`TxPipelined` is used in two repos and is the reason `ProcessPipelineHook` is
implemented rather than left out: a pipeline that fires one `duration` sample per
member command would drown the histogram, so it records one `pipeline` sample and
one `pipeLen` sample per round trip.

### What a service writes

```go
cache, err := redis.Open(ctx, log, cfg.Redis)
if err != nil {
	return err
}
monkit.Default.ScopeNamed("redis").Chain(cache)

group.Add(lifecycle.Item{Name: "redis", Close: cache.Close}) // Run is nil

val, err := cache.Get(ctx, "key").Result()
if errors.Is(err, goredis.Nil) { /* miss */ }
```

`Close` alone is the documented shape for a connection pool
(`lifecycle/lifecycle.go:43`, `docs/ADOPTING.md:54`), which currently has no
concrete example in the repo — and no repo in the estate closes its Redis client
today.

---

## 2. `rabbitmq/`

Files: `rabbitmq/common.go` (`Error`, `mon`), `rabbitmq/rabbitmq.go` (package
doc, `Config`, `URL`, `amqp()`), `rabbitmq/conn.go` (`Conn`),
`rabbitmq/topology.go`, `rabbitmq/publisher.go`, `rabbitmq/consumer.go`,
`rabbitmq/message.go`, tests as below, `rabbitmq/README.md`.

Four layers, each usable without the one above it: `Conn` supervises a
connection, `Topology` declares what a channel needs, `Publisher` sends, and
`Consumer` receives. A service that needs something none of them express drops to
`Conn.Session` and a raw `*amqp.Channel`.

### 2.1 `Conn` — the supervised connection

```go
type Config struct {
	Name string `help:"connection name shown in the broker UI and used to tag metrics" default:"aioz"`

	URL      string `help:"amqp:// or amqps:// URL; when set it overrides address, vhost, username and password" secret:"optional"`
	Address  string `help:"broker address as host:port" default:"127.0.0.1:5672"`
	Vhost    string `help:"virtual host" default:"/"`
	Username string `help:"broker username" default:"guest"`
	Password string `help:"broker password" secret:"optional" devDefault:"guest" releaseDefault:""`

	TLS           bool `help:"connect with TLS (amqps)" default:"false"`
	TLSSkipVerify bool `help:"do not verify the broker certificate; test use only" default:"false" hidden:"true"`

	DialTimeout time.Duration `help:"timeout for one connection attempt" default:"10s"`
	Heartbeat   time.Duration `help:"heartbeat interval; 0 uses the broker's" default:"10s"`
	ChannelMax  int           `help:"maximum channels on the connection; 0 means the protocol maximum" default:"0"`
	FrameSize   int           `help:"maximum frame size in bytes; 0 means unlimited" default:"0"`

	ReconnectDelay    time.Duration `help:"wait after a failed attempt before retrying" default:"1s"`
	ReconnectMaxDelay time.Duration `help:"upper bound on the exponential reconnect backoff" default:"30s"`

	RequireOnStart bool `help:"fail startup if the first connection attempt fails, instead of retrying in the background" default:"false"`
}

// URL builds the amqp:// or amqps:// URL. It contains the password: never log
// the result. Log Address, Vhost and Username instead, which is what this
// package does.
func (config Config) URL() string
func (config Config) amqp() (amqp.Config, error)
```

`Password` carries the sanctioned dev pair, because a local broker really is
`guest`/`guest`. `releaseDefault:""` states in the tag that nothing ships.
`ChannelMax` is `int` because amqp's `uint16` is not a bindable type; `amqp()`
range-checks `0 <= v <= 65535`. `Heartbeat` defaults to 10s because **no repo
configures one today** — every wrapper calls bare `amqp.Dial`, which is why a
half-open TCP connection can go unnoticed. `amqp091` has no `DialContext`, only
`Dial`/`DialConfig` (`connection.go:227,273`), so `DialTimeout` is enforced
through `amqp.Config.Dial: amqp.DefaultDial(DialTimeout)`.

`RequireOnStart` defaults to **false**, the opposite of redis, because `Run`
supervises: a broker briefly down at deploy time is the normal case this package
exists for, and crashlooping on it converts a self-healing condition into an
outage.

```go
// Conn is a supervised AMQP connection. It is not connected until Run is
// called, and it reconnects for as long as Run is running.
type Conn struct {
	log    *slog.Logger
	config Config
	url    string
	name   monkit.SeriesTag

	mu    sync.Mutex
	conn  *amqp.Connection
	ready chan struct{} // closed while conn is live; replaced on disconnect
	gen   int64         // connection generation, for metrics and logs

	done      chan struct{}
	closeOnce sync.Once
}

func New(log *slog.Logger, config Config) *Conn
func NewWithURL(log *slog.Logger, url string, config Config) *Conn

func (conn *Conn) Run(ctx context.Context) error
func (conn *Conn) Close() error
func (conn *Conn) Wait(ctx context.Context) (*amqp.Connection, error)
func (conn *Conn) Channel(ctx context.Context) (*amqp.Channel, error)
func (conn *Conn) Session(ctx context.Context, fn func(context.Context, *amqp.Channel) error) error
func (conn *Conn) Connected() bool
func (conn *Conn) Health(ctx context.Context) error
func (conn *Conn) Stats(cb func(key monkit.SeriesKey, field string, val float64))
```

`Run` dials, watches, and redials with exponential backoff until ctx is done or
`Close` is called. A context cancellation is how a service is asked to stop, so
`Run` returns nil on one; it returns an error only when `RequireOnStart` is set
and the first attempt fails.

The loop body is where the reference implementations' bugs are structurally
excluded:

```go
func (conn *Conn) runOnce(ctx context.Context) error {
	c, err := amqp.DialConfig(conn.url, amqpConfig)
	if err != nil {
		return Error.Wrap(err)
	}

	// Buffered on purpose. amqp091 sends the close reason on this channel
	// during shutdown and blocks if nobody is receiving, so an unbuffered one
	// deadlocks the connection it is watching.
	closed := c.NotifyClose(make(chan *amqp.Error, 1))

	conn.publish(c)       // sets conn.conn, closes conn.ready, gen++
	defer conn.retract(c) // clears conn.conn, replaces conn.ready

	select {
	case <-ctx.Done():
		return errs2.IgnoreCanceled(c.Close())
	case <-conn.done:
		return c.Close()
	case reason := <-closed:
		return Error.Errorf("connection closed: %w", reason)
	}
}
```

`closed` is a **local variable per attempt, never a struct field**, so hub's race
cannot be written. `Close` never runs under the lock a caller already holds, so
hub's self-deadlock cannot be written either. Backoff uses `time2.Sleep(ctx, d)`
(`time2/context.go:39`), doubling from `ReconnectDelay` to `ReconnectMaxDelay`
and reset on a successful connect, so a test advances a `time2.Machine` instead
of sleeping through 30 seconds. It retries indefinitely: a broker down for ten
minutes is an outage to ride out, not a reason to exit, and hub's attempt cap is
the behaviour being removed.

`Health` opens a channel rather than checking `IsClosed`: a TCP connection whose
broker has stopped accepting channels is not usable and `IsClosed` does not say
so — which is precisely what aioz-stream's `IsClosed` poll gets wrong.

`Session(ctx, fn)` runs `fn` with a channel on the live connection and runs it
again on the next connection every time the current one dies. The ctx handed to
`fn` is canceled when its channel dies. An error from `fn` after its connection
died is a reconnect and is retried; any other error is the caller's own failure
and is returned rather than retried forever. `Publisher` and `Consumer` are both
built on it, and it stays exported as the escape hatch for the service-specific
shapes listed as out of scope.

### 2.2 `topology.go` — declare it once, re-declared on every reconnect

Three near-identical DLX/DLQ blocks exist today (crawler-service
`rabbitmq.go:134-179` **and** `consumer.go:145-180` — duplicated within one repo
— plus map/crawler and parser-service `consumer.go:92-115`), and every wrapper
re-declares topology on reconnect because a new connection has none of the
previous one's channel state.

```go
type Exchange struct {
	Name       string `help:"exchange name; empty uses the default exchange"`
	Kind       string `help:"exchange type: direct, topic, fanout or headers" default:"topic"`
	Durable    bool   `help:"survive a broker restart" default:"true"`
	AutoDelete bool   `help:"delete when the last queue unbinds" default:"false"`
	Args       amqp.Table
}

type Queue struct {
	Name       string `help:"queue name"`
	Durable    bool   `help:"survive a broker restart" default:"true"`
	AutoDelete bool   `help:"delete when the last consumer disconnects" default:"false"`
	Exclusive  bool   `help:"restrict to this connection" default:"false"`
	Args       amqp.Table
}

type Binding struct {
	Exchange   string
	Queue      string
	RoutingKey string
	Args       amqp.Table
}

// DeadLetter describes a broker-side dead-letter path. When Exchange is set,
// Declare creates the dead-letter exchange, queue and binding, and adds
// x-dead-letter-exchange and x-dead-letter-routing-key to every main queue that
// does not already set them.
type DeadLetter struct {
	Exchange   string `help:"dead-letter exchange name; empty disables dead-lettering"`
	Queue      string `help:"dead-letter queue name" default:""`
	RoutingKey string `help:"routing key used when a message is dead-lettered" default:""`
	TTL        time.Duration `help:"message TTL on the dead-letter queue; 0 keeps messages forever" default:"0"`
}

type Topology struct {
	Exchanges  []Exchange
	Queues     []Queue
	Bindings   []Binding
	DeadLetter DeadLetter
}

// Declare declares everything in the topology on ch, in dependency order:
// dead-letter exchange and queue first, then exchanges, queues, bindings. It is
// idempotent against a broker that already has them, and it is called again on
// every reconnect, which is the only way a re-created broker comes back usable.
func (topology Topology) Declare(ctx context.Context, ch *amqp.Channel) error

// Validate reports a topology that cannot work - a binding naming a queue that
// is not declared, a dead-letter queue with no exchange - before anything is
// sent to the broker.
func (topology Topology) Validate() error
```

`Declare` returning an error on a **`PRECONDITION_FAILED`** deserves its own
README section: re-declaring an existing queue with different arguments kills the
channel, and since every wrapper re-declares on reconnect, a changed durability
or TTL turns into a reconnect loop that looks like a network problem. The error
is wrapped to say so.

`Args` is `amqp.Table`, which is not bindable by the `config` package. A
`Topology` is therefore built in code, not bound from a config file; the queue
and exchange **names** live in the service's own config where they already do
(crawler-service `config.go:5-39`, parser-service `internal/config/config.go:91-100`).

### 2.3 `publisher.go`

```go
type PublisherConfig struct {
	Exchange   string `help:"exchange to publish to; empty is the default exchange" default:""`
	RoutingKey string `help:"default routing key when Publish does not set one" default:""`

	Confirms       bool          `help:"wait for a broker confirm on every publish" default:"true"`
	ConfirmTimeout time.Duration `help:"how long to wait for a confirm" default:"10s"`
	Mandatory      bool          `help:"return a message the broker cannot route, instead of dropping it" default:"false"`
	Persistent     bool          `help:"mark messages persistent so they survive a broker restart" default:"true"`

	Retries    int           `help:"publish attempts after a channel failure; 0 disables retrying" default:"3"`
	RetryDelay time.Duration `help:"wait before the first retry; doubles up to the reconnect maximum" default:"200ms"`
}

// Message is what a Publisher sends and a Consumer receives.
type Message struct {
	RoutingKey  string
	Body        []byte
	ContentType string
	Headers     amqp.Table
	MessageID   string
	Correlation string
	Priority    uint8
	Expiration  time.Duration
}

type Publisher struct { /* conn, config, log, name, channel state */ }

func NewPublisher(conn *Conn, log *slog.Logger, config PublisherConfig, topology Topology) *Publisher

func (p *Publisher) Run(ctx context.Context) error
func (p *Publisher) Close() error
func (p *Publisher) Publish(ctx context.Context, msg Message) error
func (p *Publisher) PublishJSON(ctx context.Context, routingKey string, v any) error
func (p *Publisher) Stats(cb func(key monkit.SeriesKey, field string, val float64))
```

`Run` holds a `Conn.Session` that keeps one confirming channel alive, declares
`topology` on it, and re-declares on every reconnect. `Publish` waits for that
channel to exist (bounded by ctx), so a publish during a reconnect blocks rather
than failing — replacing crawler-service's `ready`+`sync.Cond`+100ms-poll gate
(`worker.go:15`) and hub's lazy reconnect-before-publish (`publisher.go:55`).

**Confirms are correlated per publish, which is the hub bug fixed by
construction.** The implementation uses
`ch.PublishWithDeferredConfirmWithContext`, which returns a
`*amqp.DeferredConfirmation` bound to that publish's `DeliveryTag`; the publisher
then waits on that value. There is no shared `notifyConfirm` channel for a
concurrent publisher to steal from. This is also why `Publish` is safe to call
from many goroutines — the channel is guarded by a mutex for the write, and the
wait happens outside it.

Retry: on a channel-level failure the publish is retried up to `Retries` times,
each attempt waiting for the session's next channel with `time2`-backed backoff.
`ctx` bounds the whole thing. A broker `NACK` is **not** retried — the broker
rejected the message, so a retry sends it again to be rejected again; it returns
an error naming the delivery tag. `Mandatory` publishes register a
`NotifyReturn`, and an unroutable message returns an error rather than
disappearing, which no current wrapper does.

`Persistent` and `ContentType: application/json` become defaults rather than
something each caller remembers, so aioz-stream's transient publishes
(`main.go:153`, no `DeliveryMode`) cannot recur. `PublishJSON` marshals and sets
`ContentType`; a `MessageID` is generated with the repo's `uuid` package when the
caller leaves it empty, which hub does by hand (`publisher.go:66`).

Metrics: `rabbitmq_publish` duration, `rabbitmq_publish_failed`,
`rabbitmq_publish_nacked`, `rabbitmq_publish_returned`, `rabbitmq_publish_retry`,
all tagged by `Config.Name`. **This reverses an earlier decision.** Per-message
metrics were previously out on the grounds that `amqp091` has no hook seam and
the caller owns the channel; now that the package owns publish and consume, it
owns the only place the estate can get queue instrumentation, and today there is
none anywhere.

### 2.4 `consumer.go`

Crawler-service `consumer.go:91-125` and parser-service `dispatcher.go:64-98` are
the same bounded-pool-plus-drain code written twice. This is that code, once.

```go
type ConsumerConfig struct {
	Queue       string `help:"queue to consume from"`
	ConsumerTag string `help:"consumer tag reported to the broker; empty generates one" default:""`

	Prefetch    int  `help:"unacknowledged messages the broker may send; values below 1 are treated as 1" default:"16"`
	Concurrency int  `help:"handlers running at once; 0 uses Prefetch" default:"0"`
	AutoAck     bool `help:"acknowledge on delivery instead of after the handler returns; loses messages on a crash" default:"false"`

	RequeueOnError bool          `help:"requeue a message whose handler failed, instead of dead-lettering it" default:"false"`
	DrainTimeout   time.Duration `help:"how long to let in-flight handlers finish at shutdown" default:"30s"`
}

// Handler processes one delivery. Returning nil acks it. Returning an error
// wrapped in Requeue nacks it with requeue set, so another consumer may take
// it. Any other error nacks it without requeue, which sends it to the queue's
// dead-letter exchange if it has one, and drops it if it does not.
//
// The context is canceled when the consumer is shutting down or its channel
// died. It is not canceled merely because the handler is slow.
type Handler func(ctx context.Context, msg Delivery) error

// Delivery is one received message. The embedded amqp.Delivery is the full
// API; Ack and Nack are handled by the consumer, so a handler must not call
// them.
type Delivery struct {
	amqp.Delivery
}

func Requeue(err error) error  // wraps err so the consumer requeues
func IsRequeue(err error) bool

type Consumer struct { /* conn, config, topology, handler, log, name */ }

func NewConsumer(conn *Conn, log *slog.Logger, config ConsumerConfig, topology Topology, handler Handler) *Consumer

func (c *Consumer) Run(ctx context.Context) error
func (c *Consumer) Close() error
func (c *Consumer) Stats(cb func(key monkit.SeriesKey, field string, val float64))
```

`Run` is a `Conn.Session` whose body declares `topology`, clamps and applies
`Qos(prefetch, 0, false)`, starts `Consume`, and pumps deliveries into a bounded
worker pool: a `sem chan struct{}` of `Concurrency` and a `sync.WaitGroup`, the
shape both repos already converged on.

Shutdown is the part worth getting exactly right, and the two existing versions
disagree:

1. `ch.Cancel(consumerTag, false)` so the broker sends no more deliveries.
2. Drain the remaining `deliveries` channel, `Nack(requeue=true)` each one that
   was already sent — parser-service does this (`dispatcher.go:72-78`), crawler
   does not, and without it those messages wait for the broker's consumer timeout
   before redelivery.
3. Wait for in-flight handlers, bounded by `DrainTimeout`, then log and return.

In-flight handlers get `context2.WithoutCancellation(ctx)` re-limited by
`DrainTimeout`, so work already started finishes instead of being torn up —
crawler-service's `context.WithoutCancel` (`consumer.go:93`) with the bound it is
missing. `Requeue`/`IsRequeue` is the whole of the ack policy: it expresses
parser-service's `shouldRequeue` (`internal/worker/errors.go:50`) without the
package taking a position on which of the caller's errors are transient. A ctx
cancellation during a handler is always treated as requeue.

Metrics: `rabbitmq_deliver` duration, `rabbitmq_deliver_failed`,
`rabbitmq_deliver_requeued`, `rabbitmq_deliver_dropped`, `rabbitmq_inflight`
gauge, `rabbitmq_consumer_restarts`.

### What a service writes

```go
broker := rabbitmq.New(log, cfg.RabbitMQ)
monkit.Default.ScopeNamed("rabbitmq").Chain(broker)
group.Add(lifecycle.Item{Name: "rabbitmq", Run: broker.Run, Close: broker.Close})

topology := rabbitmq.Topology{
	Exchanges: []rabbitmq.Exchange{{Name: "events", Kind: "topic", Durable: true}},
	Queues:    []rabbitmq.Queue{{Name: "orders", Durable: true}},
	Bindings:  []rabbitmq.Binding{{Exchange: "events", Queue: "orders", RoutingKey: "data.received"}},
	DeadLetter: rabbitmq.DeadLetter{Exchange: "events.dlx", Queue: "orders.dlq"},
}

pub := rabbitmq.NewPublisher(broker, log, cfg.Publisher, topology)
monkit.Default.ScopeNamed("rabbitmq").Chain(pub)
group.Add(lifecycle.Item{Name: "publisher", Run: pub.Run, Close: pub.Close})

sub := rabbitmq.NewConsumer(broker, log, cfg.Consumer, topology,
	func(ctx context.Context, msg rabbitmq.Delivery) error {
		var order Order
		if err := json.Unmarshal(msg.Body, &order); err != nil {
			return err // malformed: dead-letter it, a retry cannot help
		}
		if err := process(ctx, order); err != nil {
			return rabbitmq.Requeue(err) // transient: let someone else try
		}
		return nil
	})
monkit.Default.ScopeNamed("rabbitmq").Chain(sub)
group.Add(lifecycle.Item{Name: "orders-consumer", Run: sub.Run, Close: sub.Close})
```

Register `broker` **before** the publisher and consumer: `lifecycle` starts
together and closes in reverse (`lifecycle/lifecycle.go:69`), so the connection
closes last.

---

## 3. Tests

### Gating: a docker probe that skips, plus an env var that forbids skipping

**No `//go:build integration` tag.** A build tag keeps the code out of the
default `go test ./...` *and* out of `golangci-lint run ./...`, so it rots
silently and lint never sees it. `testing.Short()` is no better: it inverts the
default so the plain `go test` a developer types tries docker and fails.

```go
// requireDocker skips when there is no reachable docker daemon, unless
// AIOZ_REQUIRE_DOCKER is set, which turns the skip into a failure so an
// environment that is supposed to run these tests cannot quietly stop.
func requireDocker(t *testing.T) {
	t.Helper()
	provider, err := testcontainers.NewDockerProvider()
	if err == nil {
		err = provider.Health(context.Background())
	}
	if err == nil {
		return
	}
	if os.Getenv("AIOZ_REQUIRE_DOCKER") != "" {
		t.Fatalf("AIOZ_REQUIRE_DOCKER is set but no docker daemon is reachable: %v", err)
	}
	t.Skipf("no docker daemon; set AIOZ_REQUIRE_DOCKER to make this a failure: %v", err)
}
```

Probe through `provider.Health`, not `os.Stat("/var/run/docker.sock")`, so
`DOCKER_HOST`, rootless docker, colima and podman all work. Containers via bare
`testcontainers.GenericContainer` + `wait.ForListeningPort`, the shape already
used at
`depin-workspace/hub/internal/storage/infrastructure/usage_tracker/usage_tracker_test.go:34-56`
— the estate's only existing testcontainers usage. Images `redis:8-alpine` and
`rabbitmq:4-alpine`.

**RabbitMQ's `guest` user only accepts loopback connections.** From outside the
container it is refused with `ACCESS_REFUSED`, which reads exactly like a wrong
password. The container request must set `RABBITMQ_DEFAULT_USER` and
`RABBITMQ_DEFAULT_PASS`. This costs an afternoon if it is not written down.

### Docker-free unit tests

External test package, `require` only, `t.Parallel()`, map-literal tables:
`Config.Options` URL-versus-field precedence, timeout overrides surviving a
parsed URL, TLS config construction, `Config.URL`, the `ChannelMax` range check,
`Topology.Validate` rejecting a binding to an undeclared queue, the DLX argument
injection, `Channel()` before connect, the backoff sequence against a
`time2.Machine`, `Requeue`/`IsRequeue` round-tripping through `%w`, and a
`config.Bind` test proving the secret tags do not panic, mirroring
`config/secrets_test.go:46-59`.

### Integration tests, the ones that justify the packages

- Redis: `Health` against a live server; auth failure with a wrong password;
  `RequireOnStart` false path against a dead address; two `Open` calls on
  different `DB` values not seeing each other's keys (the aioz-stream shape).
- `Conn`: stop the broker container, start it again, assert `Stats` generation
  incremented and the `Session` function ran a second time. `Close` while `Run`
  is mid-backoff returns promptly. `Wait` returns on a canceled ctx.
- `Publisher`: **concurrent publishers each get their own confirm** — the direct
  regression test for hub's ack-stealing bug: N goroutines publish, all N
  confirms observed, no publish reports success without one. A publish issued
  while the broker is down blocks and then succeeds after it returns. A `NACK` is
  not retried. A `Mandatory` publish to an unbound routing key returns an error.
- `Consumer`: prefetch honoured; a handler error dead-letters and the message
  appears on the DLQ; `Requeue` redelivers; shutdown mid-flight drains, nacks the
  undelivered remainder and loses nothing; `PRECONDITION_FAILED` from a changed
  queue argument surfaces as a clear error rather than a reconnect loop.

Assert on observed state transitions, not wall-clock durations, and bound
everything with `stats/testcontext` (`docs/ADOPTING.md:148-159`). Use the `leak`
package to prove no goroutine survives `Close` — the consumer pool and the
supervisor are exactly where one would.

### CI

`.gitlab-ci.yml` gains a third stage. The existing `test` job stays a bare
`go test ./...`, fast and docker-free; the probe skips the container tests there.

```yaml
stages: [lint, test, integration]

integration:
  stage: integration
  image: ${GO_IMAGE}
  services:
    - name: docker:27-dind
      alias: docker
      command: ["--tls=false"]
  variables:
    DOCKER_HOST: "tcp://docker:2375"
    DOCKER_TLS_CERTDIR: ""
    TESTCONTAINERS_RYUK_DISABLED: "true"
    TESTCONTAINERS_HOST_OVERRIDE: "docker"
    AIOZ_REQUIRE_DOCKER: "1"
    GOMODCACHE: "${CI_PROJECT_DIR}/.cache/go-mod"
    GOCACHE: "${CI_PROJECT_DIR}/.cache/go-build"
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  script:
    - go test -timeout 10m ./redis/... ./rabbitmq/...
```

Three things to expect: the runner must be **privileged** for dind, a
runner-config change to confirm before merging; `TESTCONTAINERS_HOST_OVERRIDE` is
needed because `container.Host()` otherwise resolves to something the job
container cannot reach; and Ryuk is disabled because the reaper binds back to the
daemon awkwardly across the service link, safe since the job is ephemeral. Add
`make test-integration` alongside the existing four targets.

---

## 4. Dependencies

Named and justified in `CHANGELOG.md`'s `### Dependencies`.

| Module | Version | Build-graph cost |
| --- | --- | --- |
| `github.com/rabbitmq/amqp091-go` | v1.14.0 | **none.** Its one requirement, `go.uber.org/goleak`, is its own test dependency and is pruned out of our build list |
| `github.com/redis/go-redis/v9` | v9.22.0 | five modules, every one a leaf: `cespare/xxhash/v2`, `zeebo/xxh3`, `klauspost/cpuid/v2`, `go.uber.org/atomic`, `golang.org/x/sys` (already present) |
| `github.com/testcontainers/testcontainers-go` | v0.40.0 | **heavy**, test-only: Docker SDK, containerd, OpenTelemetry, `gopsutil`, roughly fifty modules |

testcontainers is the one worth arguing, and it is acceptable on two grounds. **A
consumer never sees it:** with module-graph pruning, test-only dependencies of an
imported package do not enter the importing module's build list, `go.sum` or
`go mod tidy` output. That is a different bill from the one `base58` was vendored
to avoid, which was a *build* dependency. **And there is no alternative for
RabbitMQ:** `miniredis` could replace the redis container but models no pool, no
pipelines, no TLS and none of the failure modes here, and no credible fake AMQP
broker exists. The choice is testcontainers or no integration test, and the
package's value is reconnect and redelivery behaviour that only a real broker
restart exercises. hub already depends on it for the same reason.

Do **not** add `testcontainers-go/modules/redis` or `.../modules/rabbitmq`, each
a separate module with its own version to track. Bare `GenericContainer` is two
lines longer and is needed anyway for the custom RabbitMQ user.

testcontainers pulls `sirupsen/logrus`, which `depguard` denies. That is
transitive; depguard inspects imports in our own files, so it does not fire.

Total: **2 new direct build dependencies (5 leaf transitives), 1 new direct test
dependency.** Run `go mod tidy` and confirm nothing else appears as direct.

---

## 5. Docs

**`redis/README.md`** and **`rabbitmq/README.md`** in the house skeleton
(`cycle/README.md`, `lrucache/README.md` are the models): title, one-line what, a
why-it-exists paragraph naming the divergent implementations this replaces, the
import block, `## Usage` with a composition-root snippet, claim-titled gotcha
sections, and an `## API` table.

Redis gotchas: *`redis.Nil` is a miss, not an error*; *Both packages are called
redis* (the `goredis` alias); *One pool per logical DB*; *No cluster support, and
why*.

RabbitMQ gotchas: *Your topology is re-declared on every reconnect, so changing a
queue argument breaks it* (`PRECONDITION_FAILED`); *A channel does not outlive its
connection*; *A NACK is not a retry*; *Return `Requeue(err)` for transient
failures and a bare error for poison messages*; *What this package does not do* —
the out-of-scope list from the Context section, so a migrating repo knows which
of its code must survive.

**Root `README.md`**: two rows in the component table (`README.md:11-38`) after
`lrucache`, two lines in the Layout tree (`README.md:44-70`), and a paragraph
after the `logging`/`logger` note, since this is the first pair of packages that
wrap a third-party client:

> `redis` and `rabbitmq` are the two packages here that wrap someone else's
> client, and they stop at different lines, on purpose. go-redis already handles
> reconnection and pooling, so `redis` adds construction, a startup ping, metrics
> and `Close`, hands back the library's own type, and wraps not one command.
> `amqp091` handles nothing above the wire, so `rabbitmq` owns the connection
> supervisor, topology, publisher and consumer that seven services had each
> written for themselves. Neither defines a Cache or Queue interface: the moment
> there is one, every feature the underlying library grows has to be re-exported
> through it, and services start choosing between the abstraction and the thing
> it abstracts.

**`CHANGELOG.md`**: prose under `### Added` for both packages and the dependency
accounting under `### Dependencies`, in `v0.3.0 (unreleased)`.

**`docs/ADOPTING.md`**: `redis` is the concrete example the "supplies `Close`
alone" sentence at line 54 currently lacks; `rabbitmq` is a `Run`+`Close` item.
Add both to the step-4 wiring snippet.

---

## 6. Order of work

1. `go get github.com/redis/go-redis/v9@v9.22.0` and
   `github.com/rabbitmq/amqp091-go@v1.14.0`.
2. `redis/common.go`, `redis/redis.go` (Config and `Options` first, pure and
   testable with no docker), `redis/monkit.go`.
3. `redis/redis_test.go`, docker-free table tests. Green before anything else.
4. `go get github.com/testcontainers/testcontainers-go@v0.40.0`;
   `redis/docker_test.go` (shared endpoint helper, `requireDocker`);
   `redis/client_test.go`.
5. `redis/README.md`. Redis is now complete and shippable on its own.
6. `rabbitmq/common.go`, `rabbitmq/rabbitmq.go` (Config, `URL`, `amqp()` with the
   `ChannelMax` range check), `rabbitmq/topology.go` — all pure, all testable
   without a broker.
7. `rabbitmq/rabbitmq_test.go` and `topology_test.go`, docker-free.
8. `rabbitmq/conn.go`. The `ready`-channel generation dance and the buffered
   `NotifyClose` are the two places to get exactly right.
9. `rabbitmq/docker_test.go` (non-`guest` user) and `conn_test.go`, including
   stop/start reconnect, `Close` mid-backoff, `Wait` on a canceled ctx.
10. `rabbitmq/message.go` and `rabbitmq/publisher.go`. Deferred confirmations are
    the load-bearing detail.
11. `rabbitmq/publisher_test.go`, including the concurrent-confirm regression
    test.
12. `rabbitmq/consumer.go`, then `consumer_test.go` — ack policy, DLQ routing,
    drain, `leak` check.
13. `rabbitmq/README.md`.
14. `.gitlab-ci.yml` integration stage, `make test-integration`.
15. Root `README.md`, `CHANGELOG.md`, `docs/ADOPTING.md`.

Steps 1-5 are a self-contained increment; if the rabbitmq half runs long, redis
ships without it.

### Anticipated lint friction

- `errcheck`'s `exclude-functions` covers `(io.Closer).Close` but not
  `(*amqp.Channel).Close` or `(*amqp.Connection).Close`. Write
  `defer func() { _ = ch.Close() }()` rather than widening the lint config.
- `gocritic`'s `commentedOutCode` may flag indented code inside doc comments. If
  it does, move the example into an `Example` function in a `_test.go`, which is
  better documentation anyway and has precedent in `currency/example_test.go`,
  `jwt/example_test.go` and `secret/example_test.go`.
- `Publisher.Publish` and the consumer pump will draw `gocyclo`/`funlen`
  attention. Split the retry loop and the drain into named helpers rather than
  adding a `nolint`.

---

## Verification

1. `make fmt && make lint`. Lint is blocking in CI.
2. `make test`. Unit tests, no docker, must stay fast, and the container tests
   must report as **skipped**, not failed.
3. `docker info` (daemon confirmed present locally, 29.7.2), then
   `AIOZ_REQUIRE_DOCKER=1 go test -v -race ./redis/... ./rabbitmq/...`, which
   proves the tests are not silently skipping. `-race` matters: the races in
   aioz-stream and hub are the class of bug this package exists to remove. Watch
   the reconnect test stop and restart the RabbitMQ container.
4. The config proof, a security rule rather than a bug: a scratch program that
   binds a struct embedding both `Config`s, runs `SaveConfig`, and shows
   `redis.password`, `redis.url` and `rabbitmq.url` written blank; then flip
   `Password` to `secret:"true"` with a `default` and confirm `Bind` panics as
   `config/secrets.go:39-66` promises.
5. **Sufficiency check against the real consumers**, which is what decides whether
   this achieved its purpose. Without changing those repos, write down for each of
   parser-service, crawler-service and hub-worker-manager which of its wrapper's
   functions maps to which call here, and confirm the only survivors are the
   documented out-of-scope items. If anything else has no equivalent, the API is
   not finished.
6. Consumer smoke test: from a scratch module, `go get` this module at the branch,
   wire redis, a `Conn`, a `Publisher` and a `Consumer` into a `lifecycle.Group`,
   publish and consume a message, send SIGINT, and confirm the group logs every
   component closing in reverse order with no goroutine left.
