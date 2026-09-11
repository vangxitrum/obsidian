---
type: decision
tags: [ci, lint, golangci, gitlab, aioz-stream]
created: 2026-08-17
agent: main
---

aioz-stream now lints with golangci-lint v2.12.2, ported from `aioz-map/parser-service`
(see [[parser-service-lint-setup]]). Branch `feat/lint`, based on `origin/main` (29ac7baf).
Went 460 findings -> 0, then `allow_failure: false`.

Repo-specific things that had to change vs the parser-service original:

- **`run.build-tags: [purego]`** - `make build` uses `-tags purego`, so lint must analyse the
  same build configuration.
- **MR pipelines had to be enabled.** The old `workflow:` block only allowed `develop*` and
  `main`, so a merge request started no pipeline at all and nothing could gate it. Added
  `merge_request_event` plus the `$CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS -> never`
  duplicate guard.
- **Private modules**: the lint job needs the same `GOPRIVATE=10.0.0.50/` +
  `CI_JOB_TOKEN insteadOf` block `build-job` already had. parser-service needs none.
- **Go image** bumped `registry:5000/go:1.23` -> `:1.25` (go.mod is `go 1.25`).
- **Submodules** (`mediamtx`, `w3streamcore`) stay unchecked out: `GIT_SUBMODULE_STRATEGY:
  none`, and both paths are in the exclusion list. No SSH key needed in CI.

**Manual step, still outstanding:** GitLab -> Settings -> Merge requests -> tick
"Pipelines must succeed". Without it `allow_failure: false` is decorative.

**ST1005 is scoped, not disabled**: `response.NewHttpError(code, err)` falls back to
`err.Error()` as the API message, so 116 error strings under `pkg/v1/services/`,
`internal/controllers/` and `internal/utils/image/` are the text the frontend displays.

**md5 (G401/G501) stays**: all 5 sites are transfer checksums compared against a digest the
peer computed (`BlockMetadata.Checksum`, `payload.Hash`) - changing it is a protocol change.
Unlike parser-service, none of it is on an auth path.

**G301/G302 excluded for the media dirs on purpose**: `./input`, `./input-live`, `./output`
are bind mounts shared with the grpc and livestream containers, so 0750/0600 breaks playback
when they run under a different uid. Only the internal-only paths were tightened.

Behaviour changes worth remembering: pprof moved to `127.0.0.1:6060` (remote profiling now
needs `ssh -L 6060:127.0.0.1:6060`), and `internal/core` uses `exec.CommandContext`, so a
cancelled transcode kills its ffmpeg child.

Pre-existing test failures unrelated to any of this: `summarizationclient` and
`transcribe_client` tests call the live host `aiozstreamai.tunnel.zvault.ai` and fail on TLS
from a dev machine. Also fixed a genuine compile error on `main`: `seeds/seed.go:34` passed a
`string` into `GetMediaListInput.Status` (`[]string`) - `go build ./...` had been broken
because CI only ever compiled `./cmd/http`.
