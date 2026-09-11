# aioz-template: Go service template scaffold

## Context

`/home/tuan/work/templates/aioz-template` is an empty repo (stock GitLab README only, one `Initial commit`). Its siblings in `/home/tuan/work/templates/` are three finished, reusable libraries that nothing composes yet:

| repo dir | module path | what it gives |
|---|---|---|
| `aioz-logger` | `gitlab.internal/tuan.quang.tran/aioz-log` | async `slog.Handler`: file rotation, Graylog/GELF, per-level sampling, ctx attrs |
| `aioz-config` | `gitlab.internal/tuan.quang.tran/aioz-config` | struct-tag driven cobra+viper binding, config generation, flag>env>secrets>config>default |
| `aioz-stats` | `gitlab.internal/tuan.quang.tran/aioz-stats` | monkit debug server (`/metrics`, `/health`, `/debug/pprof`, `/version/`, `/logging`), `version.Info`, `testcontext` |

Every new service currently re-derives the same bootstrap: build the logger from loaded config, wire a dynamic log level, stand up a debug/metrics port, run a business HTTP server, shut both down gracefully. This repo becomes the canonical starting point so a new service is `gonew` + edit the config struct + write handlers.

**Outcome:** a buildable, lint-clean, testable Go 1.25 service template with `run` / `setup` / `version` commands, an echo HTTP server, the aioz-stats debug server on a second port, Docker + GitLab CI, and a `gonew` instantiation flow.

## Decisions (confirmed with the user)

1. **HTTP framework:** `github.com/labstack/echo/v4` (v4.15.4 is already in the local module cache) for the business server. The debug server stays stdlib — it comes from aioz-stats as-is.
2. **Dependency wiring:** plain `require` on `gitlab.internal/...`. No tags exist on any lib, so these resolve to pseudo-versions off `main`. `GOPRIVATE=gitlab.internal/*,10.0.0.50/*` is already set in the user's `go env`; CI and the README must set it too. **No `replace` directives** — they would break `gonew` consumers.
3. **Scope:** aioz-stats debug server, Dockerfile + `.dockerignore`, lint/CI/Makefile derived from the siblings and extended with `build`, a `setup` command, and an example `config.yaml`.
4. **Instantiation:** `gonew`-compatible.
5. **Logger module-path fix (blocker, verified):** `aioz-logger/go.mod` declares `module gitlab.internal/tuan.quang.tran/aioz-log`, but `git ls-remote ssh://git@gitlab.internal/tuan.quang.tran/aioz-log.git` returns *"The project you were looking for could not be found"*. Fix the **logger repo's** `go.mod` to `gitlab.internal/tuan.quang.tran/aioz-logger` (and its README import lines), commit and push, before the template can resolve it.
6. **Version variable:** a local `internal/version` package owned by the template, so `gonew` rewrites its import path. Bootstrap copies the values into `aioz-stats`'s `version.Build` so the debug `/version/` endpoint agrees.

## gonew facts that shape the design

Read from `golang.org/x/tools@v0.48.0/cmd/gonew/main.go`:

- gonew runs `go mod download -json <srcmod>@latest`, then copies the **module cache** tree. Dotfiles survive the module zip (verified against `cobra@v1.9.1` and `echo/v4@v4.15.4`), so `.golangci.yml`, `.gitlab-ci.yml`, `.gitignore`, `.dockerignore` all come along. `.git` does not.
- It rewrites **only**: import paths inside `.go` files, the root package name, and the `module` line of `go.mod`. It does **not** touch Makefile, Dockerfile, CI YAML, or README.
- Consequence: **never hardcode the module path outside `.go` files.** The Makefile and Dockerfile must derive it with `MODULE := $(shell go list -m)` and pass it into the ldflags. Same for the binary name: `BINARY := $(shell basename $(MODULE))`.
- Target dir must be empty or absent.

## Target tree

```
aioz-template/
├── go.mod                       module gitlab.internal/tuan.quang.tran/aioz-template, go 1.25.0
├── go.sum
├── main.go                      package main: thin — internal/cmd.Execute
├── Makefile                     lint / lint-fix / fmt / test / build / docker
├── Dockerfile                   multi-stage, distroless static
├── .dockerignore
├── .gitignore                   .cache/ golangci-lint-report.xml bin/
├── .golangci.yml                copied from aioz-stats verbatim
├── .gitlab-ci.yml               lint + test + build stages
├── README.md                    rewritten: gonew usage, endpoints, config, prerequisites
├── config.example.yaml          committed sample of what `setup` generates
└── internal/
    ├── cmd/                     root.go, run.go, setup.go, version.go, common.go
    ├── version/version.go       plain string vars (ldflags target) + Register()
    ├── config/config.go         Config struct + LoggerConfig -> logger.Config mapping
    ├── logging/logging.go       build *logger.Handler + *slog.LevelVar from config
    ├── httpserver/
    │   ├── server.go            echo server: New / Run(ctx) / Close
    │   ├── middleware.go        request id, recover, slog request logger, monkit task
    │   ├── routes.go            /healthz, /readyz, /api/v1/hello
    │   └── server_test.go
    └── service/service.go       composes logger + debug server + http server under errgroup
```

## Key files

### `internal/version/version.go`

**Correction, verified empirically:** `-ldflags -X pkg.Build.Release=v` is a **silent no-op**. `cmd/link`'s `addstrdata1` splits at the last dot and looks up the symbol `pkg.Build.Release`, which does not exist (only `pkg.Build` does, and it is a struct). Test program output: `struct="" plain="PLAINVAL"`. The aioz-stats README documents ldflags that have never worked.

So `internal/version` holds **plain unexported string vars**, and publishes them into aioz-stats' struct at runtime:

```go
var (
	release = "dev"
	build   = "unknown"
	sha     = "unknown"
)

func Info() statsversion.Info { return statsversion.Info{Release: release, Build: build, SHA: sha} }
func IsRelease() bool         { return release != "dev" }
func Register()               { statsversion.Build = Info() }  // called from main, before anything reads it
```

`Register()` is what makes the debug server's `/version/` agree with the `version` command. `IsRelease()` drives `config.SetRelease`, selecting the `releaseDefault` tags. The e2e test asserts `/version/` reflects the ldflags, so this trap cannot silently regress.

### `internal/config/config.go`

Rules that constrain this struct, from aioz-config:

- Supported field types only: `int, int64, uint, uint64, float64, string, bool, time.Duration, []string`, nested structs, arrays of structs, `pflag.Value`. Anything else **panics at bind time** — so no `slog.Level`, no `map`. Log level and format are `string` fields parsed in `internal/logging`.
- Embedded structs must be **exported types**.
- `debug.Config` from aioz-stats already carries `help`/`default`/`hidden`/`releaseDefault`/`devDefault` tags — embed it as a named field so its flags land under `debug.*`.

```go
type Config struct {
	Server ServerConfig
	Log    LogConfig
	Debug  debug.Config
}

type ServerConfig struct {
	Address         string        `help:"address the HTTP server listens on"       default:"127.0.0.1:8080" user:"true"`
	ReadTimeout     time.Duration `help:"maximum duration for reading a request"   default:"15s"`
	WriteTimeout    time.Duration `help:"maximum duration for writing a response"  default:"30s"`
	IdleTimeout     time.Duration `help:"keep-alive idle timeout"                  default:"120s"`
	ShutdownTimeout time.Duration `help:"grace period for in-flight requests"      default:"20s"`
	BodyLimit       string        `help:"maximum request body size"                default:"1M"`
}

type LogConfig struct {
	Level      string `help:"log level: debug, info, warn, error"   default:"info"  user:"true"`
	Format     string `help:"log output format: text or json"       default:"text"  releaseDefault:"json" devDefault:"text"`
	AddSource  bool   `help:"include caller file:line in records"    default:"false"`
	BufferSize int    `help:"async record buffer capacity"           default:"4096"`

	File    LogFileConfig
	Graylog LogGraylogConfig
}

type LogFileConfig struct {
	Path          string        `help:"log file path; empty disables file output" default:""     path:"true"`
	MaxSize       int64         `help:"rotate after this many bytes"              default:"104857600"`
	MaxBackups    int           `help:"rotated files to keep; 0 keeps all"        default:"7"`
	FlushInterval time.Duration `help:"how often the write buffer is flushed"     default:"5s"`
}

type LogGraylogConfig struct {
	URL         string        `help:"GELF HTTP endpoint; empty disables shipping" default:""`
	Facility    string        `help:"GELF facility"                                default:"aioz-template"`
	Service     string        `help:"GELF host field"                              default:"aioz-template"`
	Timeout     time.Duration `help:"GELF HTTP timeout"                            default:"5s"`
	IndexFields []string      `help:"attributes forwarded as indexed GELF fields"  default:""`
}
```

Note `Debug.Addr`'s upstream default is `127.0.0.1:0` (ephemeral). The template overrides it at bind time with `config.SaveConfigWithOverride`-free approach: simply document `debug.addr` in `config.example.yaml` as `127.0.0.1:6060`, since the tag lives in aioz-stats and must not be edited from here.

Also in this package: `func (c LogConfig) Handler() (logger.Config, *slog.LevelVar, error)` doing the `string -> slog.Level` / `string -> logger.Format` parsing and returning the `*slog.LevelVar` for `/logging`.

### `cmd/root.go` — the wiring (mirrors `aioz-config/e2e_test.go:50-101`)

```go
var (
	registry  = config.New()
	cfg       appconfig.Config
	setupCfg  = &struct{ appconfig.Config }{}   // exported embedded type, per aioz-config
	configDir = config.ApplicationDir("aioz", "aioz-template")
	root      = &cobra.Command{Use: "aioz-template", SilenceUsage: true}
)

func init() {
	config.SetupFlag(nil, root, &configDir, "config-dir", configDir,
		"the directory to load the config from")
	root.AddCommand(runCmd, setupCmd, versionCmd)

	defaults := config.DefaultsFlag(root)
	registry.Bind(setupCmd, setupCfg, defaults, config.SetupMode(), config.ConfDir(configDir))
	registry.Bind(runCmd, &cfg, defaults, config.ConfDir(configDir))
}
```

`version` is deliberately **not** bound — it must work with no config file present.

### `cmd/setup.go`

```go
var setupCmd = &cobra.Command{
	Use:         "setup",
	Short:       "generate the config file",
	Annotations: map[string]string{"type": "setup"},
	RunE: func(cmd *cobra.Command, _ []string) error {
		if err := os.MkdirAll(configDir, 0o700); err != nil {
			return err
		}
		// Load first so regeneration preserves existing operator edits.
		if _, err := registry.Load(cmd); err != nil {
			return err
		}
		return registry.SaveConfig(cmd, filepath.Join(configDir, config.DefaultCfgFilename))
	},
}
```

### `cmd/run.go` + bootstrap ordering

The ordering trap: `config.Load` reports missing/broken keys through `slog`, but the real logger is built *from* what `Load` returns. Resolution — pass a deliberate bootstrap logger into the first `Load`, then install the real one:

```go
RunE: func(cmd *cobra.Command, _ []string) error {
	boot := slog.New(slog.NewTextHandler(os.Stderr, &slog.HandlerOptions{Level: slog.LevelWarn}))

	if _, err := registry.Load(cmd, config.WithLogger(boot), config.WithFailOnValueError()); err != nil {
		return err
	}

	h, level, err := logging.New(cfg.Log)   // wraps logger.New
	if err != nil {
		return err
	}
	slog.SetDefault(slog.New(h))
	defer func() {
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		_ = h.Shutdown(ctx)
	}()

	ctx, stop := signal.NotifyContext(cmd.Context(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	return service.Run(ctx, slog.Default(), cfg, level)
},
```

`root.ExecuteContext(context.Background())` in `main.go` so `cmd.Context()` is non-nil.

### `internal/service/service.go`

```go
func Run(ctx context.Context, log *slog.Logger, cfg appconfig.Config, level *slog.LevelVar) error {
	lis, err := net.Listen("tcp", cfg.Debug.Addr)
	if err != nil {
		return Error.Wrap(err)
	}

	dbg := debug.NewServerWithLevel(
		log.With("component", "debug"), lis, monkit.Default, cfg.Debug, level,
	)
	api := httpserver.New(log.With("component", "http"), cfg.Server)

	group, ctx := errgroup.WithContext(ctx)
	group.Go(func() error { return dbg.Run(ctx) })
	group.Go(func() error { return api.Run(ctx) })

	err = group.Wait()
	return errs.Combine(err, dbg.Close(), api.Close())
}
```

`var Error = errs.Tag("service")` per package, matching the aioz-stats house style. `monkit.Default` is the registry `mon = monkit.Package()` instruments into, so any `defer mon.Task()(&ctx)(&err)` in the service shows up on `/metrics` for free.

### `internal/httpserver/server.go`

The gap to close: aioz-stats' debug `http.Server` sets no timeouts (gosec G112). The echo server must:

```go
e := echo.New()
e.HideBanner, e.HidePort = true, true
e.Server.ReadTimeout = cfg.ReadTimeout
e.Server.ReadHeaderTimeout = cfg.ReadTimeout
e.Server.WriteTimeout = cfg.WriteTimeout
e.Server.IdleTimeout = cfg.IdleTimeout
```

`Run(ctx)` follows the aioz-stats pattern exactly — `errgroup` with one goroutine blocking on `<-ctx.Done()` then `e.Shutdown(shutdownCtx)`, one running `e.Start(addr)` and swallowing `http.ErrServerClosed`. `Close()` is the hard version.

Middleware (`internal/httpserver/middleware.go`):

- `middleware.RequestID()`
- `middleware.Recover()`
- `middleware.BodyLimit(cfg.BodyLimit)`
- `middleware.RequestLoggerWithConfig(middleware.RequestLoggerConfig{...})` with `LogValuesFunc` emitting through `slog` — echo v4's own logger is `log`-based and **depguard bans stdlib `log`**, so this is mandatory, not stylistic.
- a small monkit middleware wrapping each handler in `mon.Task()`.
- a custom `e.HTTPErrorHandler` that logs at Error and returns a stable JSON error body.

Routes (`routes.go`): `GET /healthz` (liveness, always 200), `GET /readyz` (readiness — returns 503 until a `ready atomic.Bool` is set, so real services have the hook), `GET /api/v1/hello` sample. The debug server's own `/health` stays on the debug port; `/healthz` on the API port is what a load balancer probes.

### Makefile (module-path derived, gonew-safe)

```make
MODULE  := $(shell go list -m)
BINARY  := $(shell basename $(MODULE))
VERSION ?= $(shell git describe --tags --always --dirty 2>/dev/null || echo dev)
SHA     := $(shell git rev-parse HEAD 2>/dev/null || echo unknown)
BUILT   := $(shell date -u +%Y-%m-%dT%H:%M:%SZ)

LDFLAGS := -s -w \
  -X $(MODULE)/internal/version.release=$(VERSION) \
  -X $(MODULE)/internal/version.sha=$(SHA) \
  -X $(MODULE)/internal/version.build=$(BUILT)

build:
	CGO_ENABLED=0 go build -trimpath -ldflags "$(LDFLAGS)" -o bin/$(BINARY) .
```

plus `lint`, `lint-fix`, `fmt`, `test` copied verbatim from `aioz-stats/Makefile` (golangci-lint pinned `v2.11.4`), and a `docker` target.

### Dockerfile

Multi-stage: `registry:5000/go:1.25` builder → `gcr.io/distroless/static-debian12:nonroot`. The builder stage runs `make build` so the ldflags stay defined once. Needs `ARG GOPRIVATE=gitlab.internal/*` and an SSH/netrc mount for the private modules — document `DOCKER_BUILDKIT=1 docker build --ssh default` in the README. `.dockerignore` excludes `bin/`, `.cache/`, `.git/`, `*.md`.

### `.gitlab-ci.yml`

Start from `aioz-config/.gitlab-ci.yml` (which already has `lint` + `test`), add a `build` stage running `make build` with the binary as an artifact, and add to every job's `before_script`:

```yaml
- go env -w GOPRIVATE=gitlab.internal/*
- git config --global url."ssh://git@gitlab.internal/".insteadOf "https://gitlab.internal/"
```

`.golangci.yml` is copied from `aioz-stats` unchanged (depguard's slog-only rule is exactly what this template needs).

## Files touched / created

- **Modified (other repo, prerequisite):** `/home/tuan/work/templates/aioz-logger/go.mod` (module path), `/home/tuan/work/templates/aioz-logger/README.md` (import examples). Commit + push before the template's `go mod tidy` can succeed.
- **Created:** everything in the tree above under `/home/tuan/work/templates/aioz-template/`. `README.md` is rewritten from the GitLab stock text.

## Verification

End-to-end, in order:

```sh
cd /home/tuan/work/templates/aioz-template

# 1. resolves and builds
go mod tidy && make build

# 2. lint clean (the real gate: depguard, gocritic, revive)
make lint

# 3. version command, with ldflags actually applied
./bin/aioz-template version          # expect the git SHA, not "unknown"

# 4. config generation, and idempotency
./bin/aioz-template --config-dir /tmp/aioz-tpl setup
cat /tmp/aioz-tpl/config.yaml        # defaults commented out; server.address + log.level live (user:"true")
sed -i 's|^# server.address:.*|server.address: 127.0.0.1:18080|' /tmp/aioz-tpl/config.yaml
./bin/aioz-template --config-dir /tmp/aioz-tpl setup
grep '^server.address' /tmp/aioz-tpl/config.yaml   # edit survived regeneration

# 5. run, then probe both ports
./bin/aioz-template --config-dir /tmp/aioz-tpl run &
curl -sf 127.0.0.1:18080/healthz
curl -sf 127.0.0.1:18080/readyz
curl -sf 127.0.0.1:18080/api/v1/hello
curl -sf 127.0.0.1:6060/health
curl -sf 127.0.0.1:6060/version/     # same SHA as step 3
curl -sf 127.0.0.1:6060/metrics | head
curl -sf 127.0.0.1:6060/logging                        # INFO
curl -sf -X PUT -d DEBUG 127.0.0.1:6060/logging        # DEBUG, and log output changes live
kill %1                              # graceful: both servers log shutdown, exit 0

# 6. env + flag precedence
AIOZ_SERVER_ADDRESS=127.0.0.1:18081 ./bin/aioz-template --config-dir /tmp/aioz-tpl run  # env beats file
./bin/aioz-template --config-dir /tmp/aioz-tpl run --server.address=127.0.0.1:18082     # flag beats env

# 7. gonew round trip (the actual template contract) — after pushing the repo
cd /tmp && go run golang.org/x/tools/cmd/gonew@latest \
  gitlab.internal/tuan.quang.tran/aioz-template gitlab.internal/tuan.quang.tran/demo-svc
cd demo-svc && make build && ./bin/demo-svc version   # binary is demo-svc, ldflags path rewrote

# 8. docker
DOCKER_BUILDKIT=1 docker build --ssh default -t aioz-template:dev .
```

Automated tests to commit:

- `internal/httpserver/server_test.go` — `testcontext.New(t)`, real `httptest` requests through the echo instance for `/healthz`, `/readyz` (503 then 200), the error handler shape, and a close/run race loop mirroring `aioz-stats/debug/server_test.go:91-103`.
- `internal/config/config_test.go` — the `LogConfig -> logger.Config` mapping, including invalid level/format returning an error rather than panicking.
- `e2e_test.go` at the root — builds the binary with `go build`, runs `setup` into `t.TempDir()`, asserts the generated file content, then runs `run` on ephemeral ports and probes `/healthz` and the debug `/metrics`.

Note step 5's `--config-dir` values assume `debug.addr` is set to `127.0.0.1:6060` in the generated config; aioz-stats' own default is `127.0.0.1:0` (ephemeral), which `config.example.yaml` documents.
