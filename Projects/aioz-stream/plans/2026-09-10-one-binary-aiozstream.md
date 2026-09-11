# One binary: `aiozstream`, with a global `--config-dir`

## Context

The service shipped three binaries built from three `main` packages - `cmd/http`,
`cmd/grpc`, `cmd/migrate` - each with its wiring in a `func init()`. The goal is
one binary, `aiozstream`, with a depin-style global flag on the root command:

```
aiozstream [--config-dir DIR] api            serve the public HTTP API
aiozstream [--config-dir DIR] grpc           serve the internal gRPC API
aiozstream [--config-dir DIR] migrate up     apply the schema (also down/version/force)
aiozstream version                           print the build stamp (--json)
```

Config is read from `<config-dir>/config.yaml`, `--config-dir` defaulting to `.`.
**`APP_ENV` goes away entirely** (user decision). Today each entry point reads
`./$APP_ENV.yaml` with *different* defaults - `api`/`grpc` fall back to `debug`,
`migrate` to `app` - and that divergence is exactly the trap the migrate comment
warns about. One rule replaces it.

Env-var overrides of individual keys (`POSTGRES_HOST` etc., bound from each
field's `env:` tag in `internal/config/config.go:253`) are untouched - only the
choice of *file* changes.

## Already in place, uncommitted (do not redo)

- `cmd/aiozstream/main.go` - cobra root + local `versionCommand()` on
  `aioz-common/version`.`Build`.
- `internal/cli/{api,grpcserver,migrate}` - the three entry points, `init()`
  turned into an explicit `setup()` called from `Run()`, so `version`/`--help`
  open nothing. Compiles (`go build ./cmd/... ./internal/cli/...` exits 0).
- Makefile single `build` -> `bin/aiozstream`; Dockerfile `CMD`s; compose binary
  mounts; `.air.toml`; `.golangci.yml`; `SOURCE_STRUCTURE.md`; CI build-job.

## Changes

### 1. Global `--config-dir` flag (Go)

- **New `internal/cli/config.go`** (package `cli`): a `Globals` struct holding
  `ConfigDir`, a `ConfigFile = "config.yaml"` const, and
  `(*Globals).ConfigPath() string` = `filepath.Join(ConfigDir, ConfigFile)`.
  The subpackages import `internal/cli`; it imports none of them, so no cycle.
- **`cmd/aiozstream/main.go`**: extract `newRootCommand() *cobra.Command` from
  `main()` (mirrors `cmd.New()` in `~/work/templates/aioz-template/internal/cmd/root.go`),
  register `root.PersistentFlags().StringVar(&g.ConfigDir, "config-dir", ".",
  "directory holding config.yaml")`, pass `g` to `api.Command(g)`,
  `grpcserver.Command(g)`, `migrate.Command(g)`. `version` stays unbound: it
  must work with no config present.
  - Plain `PersistentFlags` is enough. depin's `cfgstruct.SetupFlag` scans
    `os.Args` early only because its `Bind` bakes `$CONFDIR` into struct
    defaults before parsing; here config is loaded inside `Run`, after cobra has
    parsed.
- **`internal/cli/api/wire.go:195-207`** and **`internal/cli/grpcserver/wire.go:56-68`**:
  `setup()` becomes `setup(configPath string)`; drop the `APP_ENV` block, call
  `config.MustNewAppConfig(configPath)`. `Run()` -> `Run(configPath string)`.
- **`internal/cli/migrate/migrate.go:97-106`**: `migrator()` takes the path;
  delete the APP_ENV block and its "default is app, not debug" comment (the
  divergence no longer exists).
- **`internal/cli/{api,grpcserver}/command.go:13`**: `Long` text ->
  "Reads <config-dir>/config.yaml (default ./config.yaml)".

### 2. Config file rename in the repo

- Local dev file: `debug.yaml` -> `config.yaml` at repo root. Both are already
  ignored by the `*` allowlist in `.gitignore:22`; nothing to add.
- `env-example/app.yaml` -> `env-example/config.yaml` (`git mv`; still tracked
  via `!env-example/**` at `.gitignore:69`). Update its readers:
  `internal/config/loadcheck_test.go:21,33,35`,
  `internal/config/config_yaml_test.go:37,43,61,87`, and the three comment lines
  in `env-example/app.env:4-7` that name `app.yaml`. `app.env` itself stays - it
  is the compose `env_file:` for the postgres/rabbitmq images, not the service.
- `internal/config/debug_yaml_test.go` -> `git mv` to `local_yaml_test.go`,
  test renamed `TestLocalConfigYamlLoads`, path and messages to `config.yaml`.
- `internal/seeds/seed.go:22` `"../../debug.yaml"` -> `"../../config.yaml"`;
  comments at `internal/seeds/seed_test.go:9,16`.
- Leave `internal/app/highlight/integration_test.go:57` (`test.yaml`, its own
  viper instance) alone.

### 3. Build / deploy surface

- **Makefile:90,93**: `run`/`run-grpc` drop `APP_ENV=debug` ->
  `./bin/$(BIN_NAME) api` / `grpc`.
- **`deploy/docker/{api,grpc}.Dockerfile:6`**: drop `ENV APP_ENV=app`. WORKDIR
  is `/app`, so the default `--config-dir .` reads `/app/config.yaml`.
- **`docker-compose.yml`** `api` and `grpc` services: drop `APP_ENV` + its
  comment; mount `./config.yaml:/app/config.yaml`.
- `livestream`/`livestream2` services and `live.Dockerfile` are a different
  binary (`mediamtx`, submodule not checked out here, so what it reads can't be
  verified). Keep their `APP_ENV` and their mount *target* `/app/app.yaml`, and
  change only the host source to `./config.yaml`, so they see exactly what they
  saw before from the single renamed host file.
- **`.gitlab-ci.yml` `migrate-job`** (`:313-338`), currently broken: it scp's a
  `migrate` artifact build-job no longer produces. Rewrite to:
  - scp `${API_BIN_NAME}` to `/tmp/aiozstream`, run it **from `/tmp`**, never
    install into `${DEPLOY_PATH}` - that path is the live API binary, and the
    job's own design note says the running service stays untouched until the
    deploy job rotates it in.
  - `--config-dir "$(dirname "${DEPLOY_PATH}")"` - the compose directory, the
    same file the containers mount. The old job read `${DEPLOY_PATH}/app.yaml`,
    a separate copy that could drift from the containers' config.
  - Preflight: `test -f "$CONF_DIR/config.yaml"` with an error naming the file,
    so a host that has not been renamed fails loudly before touching the
    database.
  - `dirty=true` grep still matches `migrate version`'s printf; update the
    "Fix it by hand" hint to `aiozstream --config-dir ... migrate force <version>`.

### 4. Stale prose

- `docs/LINTING.md:42,251,368` and code comments at
  `internal/app/server/monitoring.go:15`, `internal/app/server/server.go:161`,
  `internal/app/debug/boot_test.go:12`, `internal/middlewares/log_test.go:165`:
  `cmd/http` -> `internal/cli/api`.
- `SOURCE_STRUCTURE.md`: add one line under `cmd/aiozstream` for
  `--config-dir` / `config.yaml`.

### 5. Guardrail test: `cmd/aiozstream/main_test.go`

No database needed:
- Root has `api`, `grpc`, `migrate`, `version`; `migrate` has `up`, `down`,
  `version`, `force`.
- `--config-dir` is a persistent flag on root, default `.`, and is visible from
  `api`, `grpc` and `migrate up` (`cmd.Flags().Lookup` on the leaf).
- `version` and `version --json` run and produce output (the JSON parses) - in
  a test process with no config file and no Postgres, which is the regression
  the `init()` -> `setup()` move fixed.
- `Globals{ConfigDir: "/etc/aioz"}.ConfigPath()` == `/etc/aioz/config.yaml`.

### 6. Commit

One commit, conventional format, e.g.
`refactor(cli): merge binaries into aiozstream with --config-dir`. The body
calls out the deploy prerequisite below.

## Deploy prerequisite (ops, one-time per host - not done by this change)

Before the first release carrying this change, on each host, in the compose
directory (`dirname $PROD_DEPLOY_PATH`, `dirname $STAG_DEPLOY_PATH`):
`mv app.yaml config.yaml`, then pull the updated `docker-compose.yml`
(`make upload-deploy-resource`). The migrate-job preflight fails the pipeline
with a clear message if this was skipped.

## Verification

From the worktree root, each must pass before committing:

```sh
go build ./...
make test                          # includes the new main_test.go and renamed config tests
make lint                          # 0 issues
make swagger-verify
make build && ls -lh bin/aiozstream
make verify-stamp RELEASE=$(git rev-parse --short HEAD) CHANNEL=dev
grep -rn APP_ENV --exclude-dir=.git --exclude-dir=vendor .   # only live.Dockerfile + livestream compose entries remain
```

Exercise the binary:

```sh
./bin/aiozstream --help                               # lists --config-dir and the four subcommands
./bin/aiozstream api --help                           # shows inherited --config-dir
./bin/aiozstream version && ./bin/aiozstream version --json | jq .
cd /tmp && ~/…/bin/aiozstream version                 # no config.yaml anywhere: still prints
./bin/aiozstream --config-dir /nonexistent api        # fails naming /nonexistent/config.yaml
```

End-to-end against local infra (`docker compose --profile vod up -d postgres redis rabbitmq`):

```sh
mv debug.yaml config.yaml
./bin/aiozstream migrate version                      # default ./config.yaml, talks to Postgres
mkdir -p /tmp/cfg && cp config.yaml /tmp/cfg/
(cd /tmp && ~/…/bin/aiozstream --config-dir /tmp/cfg migrate version)   # proves the flag, not cwd
make run && curl -s localhost:8080/ping               # -> pong
make run-grpc && grpcurl -plaintext localhost:$GRPC_PORT list
```

Docker path: `cp config.yaml` as the compose-dir file, `docker compose build api grpc &&
docker compose up -d api grpc`; both stay up, `docker compose logs api` shows
the server binding - proves the `APP_ENV`-free images find `/app/config.yaml`.

CI `migrate-job` can't run locally; review its script by hand and, if a staging
tag is available, watch the first staging pipeline's migrate stage.
