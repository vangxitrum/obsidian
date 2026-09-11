---
title: parser-service migrated to the shared aioz-log package
type: project
updated: 2026-08-21
---

# parser-service uses the shared AIOZ logger

Done 2026-08-21. Replaced the in-house `internal/utils/customlog` with
`gitlab.internal/tuan.quang.tran/aioz-log` - the same module and the same pin
(`v0.0.0-20260701080241-ba9d50a6e7ed`) that aioz-map/crawler-service uses.

## Shape (mirrors crawler)

Only `cmd/` builds a handler; everything else takes a plain `*slog.Logger`.
`internal/utils/applog` is a thin adapter so both binaries share one set of
defaults instead of duplicating them:
- `applog.New(Options{Level, Format, Sampling}) (*slog.Logger, *aiozlog.Handler, error)`
- `applog.ParseLevel("debug"|...)`
- `applog.WithRequestID(ctx, id)` -> `aiozlog.WithContext(ctx, slog.String("request_id", id))`

The shared console handler DOES read ctx attrs set via `WithContext`, so the old
parser-specific context key was deleted outright.

**Import alias is `aiozlog`, not `logger`** - `logger` is used as a local
variable name in both mains and would shadow the package.

## Non-obvious consequences (these are the expensive part, not the code swap)

Adding ONE private module changed the build contract of the whole repo:

1. **CI**: `.gitlab-ci.yml` previously stated in its header that this repo needed
   no credentials. Now has a `.private_module_auth` hidden job (job-token
   `insteadOf` written to GLOBAL git config - local config does not apply because
   `go mod download` runs git from inside `$GOMODCACHE/cache/vcs/<hash>`), spliced
   into every Go job via `!reference`. Includes a preflight `git ls-remote` that
   fails with the exact allowlist instruction.
   MANUAL STEP: tuan.quang.tran/aioz-logger -> Settings -> CI/CD -> Job token
   permissions -> add parser-service.
2. **Dockerfiles**: both need a BuildKit secret (`--secret id=gitlab_token`) plus
   `deploy/ci/certs/gitlab-internal.crt` + `update-ca-certificates`, because the
   internal GitLab sends a leaf with no chain from an untrusted issuer. Copied the
   pattern from `crawler-service/deploy/local/crawler.Dockerfile`. TLS verify is
   deliberately NOT disabled (the token is an HTTP basic credential).
3. `GOPRIVATE=gitlab.internal/*,10.0.0.50/*` set in CI job vars and Dockerfiles.

## Behavior change worth knowing

The old customlog shortened `source` to `file:line` via `ReplaceAttr`. The shared
handler emits the **full absolute path** and its `Config` exposes no ReplaceAttr
hook, so this is now `src=/home/.../endpoint.go:52`. Same as crawler. Accepted for
consistency; preserving the short form would mean wrapping the handler, which
defeats the point of sharing it.

Sampling is available (`Options.Sampling`) but left nil: the API access log is one
record per request and sampling would silently drop them. Crawler does enable it.

Also: the ANSI color constants the gin access log uses moved from customlog into
`internal/pkg/middleware/color.go` - they are presentation for one middleware, not
logging infrastructure. `customlog.LogClient` (a Graylog seam) was dead code and
was dropped; the shared package has a real `GraylogConfig` if that is ever wanted.

Related: [[agm-selfreview-test-coverage]]

## CI test stage (added same day)

Both repos now have a reporting test stage, not just `go test`:

- `go test -v -covermode=atomic -coverprofile=coverage.out ./... | tee test-output.txt`
  wrapped in `set -o pipefail` - **without pipefail the job reports tee's exit
  status and every failing suite looks green**. Verified with a deliberately
  failing test: exit 1 and the failure named in the report.
- `after_script` (not script) converts the output, so reports still upload when
  the suite failed - that is the run whose report you want.
  `go-junit-report` -> JUnit, `gocover-cobertura` -> Cobertura. Both pinned
  (v2.1.0 / v1.3.0) in CI variables and installed with `go install` from the
  public proxy, so GOPRIVATE does not apply to them.
- `artifacts: when: always` + `reports: {junit, coverage_report}` +
  `coverage: '/^total:\s+\(statements\)\s+(\d+\.\d+)%/'`.
  Test names carry their Jira TC id, so the MR widget names the broken
  acceptance criterion directly.
- Crawler additionally got `.private_module_auth` extracted; `build` and `test`
  previously did `!reference [lint, before_script]`, which dragged in the
  golangci-lint install AND `make lint-verify` on every job.

Local numbers at the time: parser 3.3% coverage (most packages have no tests),
crawler 38.3%, 385 testcases / 4 failure entries / 10 skipped.

Not enabled: `-race`. It needs CGO_ENABLED=1 plus a C toolchain in the image,
and both jobs run CGO_ENABLED=0. Worth a separate job for the crawler, which is
goroutine-heavy.
