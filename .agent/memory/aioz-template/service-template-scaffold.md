---
type: decision
tags: [go, template, gonew, aioz, scaffold]
created: 2026-09-04
agent: main
---

`/home/tuan/work/templates/aioz-template` (module `gitlab.internal/tuan.quang.tran/aioz-template`, go 1.25.0) is the canonical starting point for a new AIOZ Go service. It composes the three sibling libraries rather than re-deriving their bootstrap: [[aioz-config]] for flags/env/config generation, `aioz-logger` for the slog handler, `aioz-stats/debug` for the metrics-pprof-version-logging port.

## Shape (user-approved 2026-09-04)

- `main.go` -> `internal/cmd/` (cobra: `run`, `setup`, `version`). No `cmd/<name>/` directory: gonew does not rename directories, so a root `main.go` avoids shipping a binary literally called `aioz-template`.
- HTTP: **echo v4** for the business API on `server.address`; the aioz-stats debug server stays stdlib on `debug.addr`.
- Deps by pseudo-version off `main` (no tags anywhere), `GOPRIVATE=gitlab.internal/*`. **No `replace` directives** - they are copied verbatim into every generated service.
- `internal/service/` is a deliberate throwaway sample domain showing the three house conventions.

## gonew contract

gonew rewrites only the `module` line in go.mod, import paths in `.go` files, and the root package name. Makefile, Dockerfile, CI YAML and README keep the literal name. So **no non-Go file hardcodes the module path**: `MODULE := $(shell go list -m)`, `BINARY := $(notdir $(MODULE))`, and the Dockerfile does `MODULE="$(go list -m)"` inline. `make check-template` no-ops in the template itself and lists leftovers in a generated service. Dotfiles do survive the module zip (checked against cobra and echo in the module cache); `.git` does not.

## Bootstrap ordering in `run`

`config.Load` logs through slog, but the real logger is built from what Load returns. So: bootstrap stderr logger -> `Load(WithLogger(boot), WithFailOnValueError())` -> `cfg.Validate()` (rejects a bad level/format before a handler is built from it) -> `logging.New` -> `slog.SetDefault` -> **replay** `LoadResult.MissingKeys`/`BrokenKeys` on the real logger so they reach the file and Graylog sinks -> `signal.NotifyContext` -> errgroup -> `handler.Shutdown` last, on `context.WithoutCancel`.

See [[go-ldflags-x-struct-field-noop]], [[aioz-config-embedded-struct-flattening]], [[aioz-stats-testcontext-check]], [[go-private-modules-in-docker]].
