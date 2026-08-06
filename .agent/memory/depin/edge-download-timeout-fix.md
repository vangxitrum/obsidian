---
type: decision
tags: [edgeserver, hang, memory-leak, goroutine-leak, timeout, grpc, incident, prometheus, k6]
created: 2026-07-22
agent: main
---

**Root-caused and fixed a live edge-server hang/memory-leak incident.** Branch
`feat/adding-log-monitor`, uncommitted (no-auto-commit rule).

## The incident (real, observed live via Prometheus)
`edge-server1` instance `70f7648399c0`, hit by a k6 load test (`testid=hls-run-001`,
`testrun_name=hls-loadtest`, scenario `viewers`, 50 concurrent VUs requesting HLS
segments) starting ~17:03. Handled fine through 17:13 (2,361 successes, 0% fail).
Then tipped over: `goroutines{role="edge-server1"}` exploded 32 -> 406,478 in ~15 min
and **never came back down** even after the test ramped down to 2 VUs at 17:25.
`runtime_memstats{field="Sys"}` (OS-reserved memory) climbed to **11.4 GB and stayed
pinned**. `edge_download_duration{field="max"}` climbed to 473s+. k6 saw
`error_code:1050 "request timeout"`, `1000 "unexpected EOF"`, and `1504 status:504`
gateway timeouts. Confirms the plateau-not-decay shape is a genuine goroutine leak,
not healthy load-proportional concurrency.

## Root cause (static trace, confirmed structurally)
No timeout anywhere in the download path past the initial 20s TCP dial:
- `edgeserver/handler.go` `handleDownload` used raw `r.Context()` (no deadline).
- `edgeserver/server.go`'s `http.Server{}` sets no `ReadTimeout`/`WriteTimeout`/`IdleTimeout`.
- go-sdk `download.go` has zero `context.With*` calls anywhere.
- `go-sdk/internal/piecedownload/manager.go` `Fetch`: `context.WithCancel` only;
  cancels the slow tail once `requiredCount` (29-of-80 RS, `cmd/coord/main.go:255-262`)
  pieces succeed - but if too many concurrent candidates stall at once (real under
  load: shared 100-conn/5-per-key dial pool, `go-sdk/internal/dial/dial.go:82-88`),
  the `<-resCh` receive loop just blocks forever.
- `go-sdk/internal/workerclient/download.go` `downloadStreamReader.Read`: once a
  worker accepts the piece stream and goes silent (no error/data/close),
  `stream.Recv()` blocks forever - no read deadline, and grpc client has **no
  keepalive params** (`go-sdk/internal/grpcutil/grpcconn/*.go` - `grpc.NewClient`
  with no `WithKeepaliveParams`), so a stalled-but-open peer is never detected.
- Each permanently-hung request pins its goroutine(s), open connection, and every
  already-fetched piece buffer (up to ~one segment's worth, 64 MiB
  `coord/file/service.go:32` default) forever - compounding across occurrences since
  nothing ever reclaims them.

## Fix - v2: moved into go-sdk itself (user's call, better than v1)
First pass put the timeout wrap in `edgeserver/handler.go` only. User pushed back:
"why don't we just let the SDK set a default download timeout and let us configure
that" - right call, since edgeserver was only ONE of go-sdk's download consumers;
every other caller of `Client.DownloadByTicket/DownloadFile/DownloadManifest` had
the identical unbounded-hang gap. Moved ownership into go-sdk, safe-by-default:

- **go-sdk** (`/home/tuan/work/depin-workspace/go-sdk`, separate git repo):
  - `options.go`: new `WithDownloadTimeout(d time.Duration) ClientOption`,
    doc'd with the same "non-positive falls back to the default" idiom the
    sibling `WithDownloadConcurrency` already uses in this exact file - reused
    the existing convention rather than inventing a `<=0 disables` one.
  - `client.go`: `Client.downloadTimeout time.Duration` field, wired through
    `New()`.
  - `download.go`: `const DefaultDownloadTimeout = 2 * time.Minute` (exported)
    + `(c *Client) withDownloadTimeout(ctx)` helper; wraps `ctx` at the top of
    both `DownloadManifest` and `DownloadManifestRange` (the two chokepoints
    every whole-file/ticket/range download entry point funnels through) - so
    **every** go-sdk download consumer is now bounded by default, not just
    edgeserver, even if they never call `WithDownloadTimeout` at all.
  - Tests: `download_internal_test.go` - `TestClient_WithDownloadTimeout_*`
    (default applies when unset, configured value wins, non-positive falls back
    to default - deterministic, checks `ctx.Deadline()` only, no goroutines).
    `internal/piecedownload/manager_test.go` -
    `TestFetch_BoundedByContextDeadline` (new): proves the deeper assumption the
    whole fix depends on - that `Manager.Fetch` actually unblocks and returns
    once an outer context deadline fires, even when `requiredCount` can never be
    reached (every worker "stalls" via `mockGetter{delay: time.Hour}` racing
    `<-ctx.Done()`, already the file's existing fake-pattern, just given an
    effectively-infinite delay). Full go-sdk suite run: all green except the
    pre-existing, already-documented `TestPieceIDScanNullAndEmpty` failure (see
    [[edgeserver-metadata-headers]]) and a pre-existing `go vet` self-assignment
    warning in `pkg/infectious/addmul_amd64.go` - neither touched by this change.

- **depin/edgeserver** (this repo) - simplified back down now that the SDK owns
  the bound:
  - `handler.go`: `handleDownload` calls `DownloadByTicket(r.Context(), ...)`
    directly again, no local context wrapping. Kept the one edgeserver-side
    piece that's still genuinely edgeserver's job regardless of where the
    timeout lives: `writeDownloadError` maps a failed download to a real HTTP
    status (504 for `context.DeadlineExceeded`, 502 otherwise) when zero bytes
    have streamed yet, instead of the pre-fix silent 200 OK + empty body.
  - `server.go`: `New()` passes `uplinksdk.WithDownloadTimeout(cfg.Server.DownloadTimeout)`
    when building the client. Dropped the local `(*Server).withDownloadTimeout`
    method (moved to go-sdk). Kept the `downloadClient` interface + `newServer`
    accepting it - still earns its keep for deterministic error-mapping tests.
  - `config.go`: `ServerConfig.DownloadTimeout` field kept (still the edge
    operator's configuration surface - "let us configure that"), but its
    cfgstruct `default:"2m"` tag was **removed** - the zero value now correctly
    delegates to the SDK's own `DefaultDownloadTimeout`, so the "2 minutes"
    magic number has exactly one source of truth instead of two that could drift.

## Tests (final, depin side)
`edgeserver/handler_test.go` was rewritten once the SDK took over bounding: the
original two goroutine/timing-based tests (`stalledClient` blocking on
`<-ctx.Done()`) no longer tested anything meaningful at the handler layer once
`handleDownload` stopped wrapping context itself, and would have hung outright
against the new code (nothing in the handler cancels the fake's ctx anymore).
Replaced with 4 deterministic, synchronous `fakeDownloadClient`-based tests
(`TestHandleDownload_DeadlineExceeded_Returns504`,
`_OtherError_Returns502`, `_ErrorAfterBytesWritten_StatusNotOverridden`,
`_Success`) - no timers, no goroutines, ~2x faster suite. **Verified the v1
regression test was real before the redesign**: manually reverted just the
timeout-wrapping line (kept the interface/plumbing) and reran - it genuinely
failed (2s timeout) against pre-fix behavior, passed after restoring the fix.
That confidence carried into moving the logic to go-sdk; the new go-sdk-level
tests (`TestFetch_BoundedByContextDeadline` etc.) are the ones now actually
verifying the "does a deadline unblock a stalled fetch" guarantee.
Full `edgeserver` suite green, `go vet`/`gofmt` clean, `cmd/edgeserver` builds.

## Gotchas (durable)
- **This treehouse worktree (`depin-b971d9/3`) had no `../go-sdk`** (the `replace`
  target) - symlinked `ln -s /home/tuan/work/depin-workspace/go-sdk
  /home/tuan/.treehouse/depin-b971d9/3/go-sdk` to build/test at all. Not a repo
  change; same pattern as prior sessions' worktrees (see
  [[edge-outbound-bandwidth]]).
- **Prometheus (`localhost:9090`) has real per-process metrics for every coord/edge
  role**, not just what you'd expect from a scrape target list (`/api/v1/targets`
  only shows `job=prometheus` self-scrape - everything else arrives via
  remote_write and only shows up via `/api/v1/label/__name__/values` /
  `/api/v1/series`). Two metric families for the "same" thing coexist: standard
  `go_goroutines`/`go_memstats_*`/`process_*` (client_golang, only ever
  Prometheus's own self-metrics here) vs monkit's `goroutines`/`runtime_memstats`/
  `runtime_gcstats` (bare names, `scope=github.com/spacemonkeygo/monkit/v3/environment`
  label, tagged by `role`/`instance`/`app`) - the monkit ones are the real
  per-service data. See also [[coord-monitoring-vps-artifacts]] re: monkit field
  naming gotchas.
- **k6 load-test metrics also land in this same Prometheus** (`k6_*` series,
  tagged `testid`/`testrun_name`/`scenario`/`name`/`error_code`) - genuinely useful
  for correlating client-observed failures (timeouts, EOF, gateway 504s) against
  server-side goroutine/memory telemetry to nail down a trigger.
- Local `fleet-worker-1`/`fleet-worker-2` docker containers in this environment are
  a red herring for this incident - they crash-loop fast on missing identity certs
  (connection-refused fast failure, not a stall), unrelated to the real fleet
  edge-server1 talks to.
- **Merge conflict with develop's ranged-download/compression/canonical-route
  work (2026-07-22, same day)**: `git merge develop` into `feat/adding-log-monitor`
  brought in the FULL edge feature set from [[edge-http-range]]/
  [[edge-outbound-bandwidth]]/[[coord-getdownloadinfo-range]]/[[gosdk-ranged-download]]
  (`serve`/`serveFull`/`serveRange`/`serveHead`, ETag/304, gzip, canonical
  `/download/{fileId}`), conflicting with my `handler.go`. Git's line-diff
  produced a deceptive conflict shape: my inline `setHeaders` closure (inside
  `handleDownload`) is textually near-identical to develop's extracted
  `metadataHeaderSink` function, so diff3 aligned them as "common" context and
  split the REAL conflict oddly across two disjoint hunks. Resolved by reading
  both full pre-merge versions (`git show HEAD:path` / `git show
  MERGE_HEAD:path`) rather than trusting the marked hunks, keeping develop's
  full file as the base and layering my `writeDownloadError` (504/502 mapping)
  onto `serveFull`'s error branch only - NOT `serveRange`'s, since that path
  already calls `cw.WriteHeader(http.StatusPartialContent)` before the transfer
  even starts, so the status can't be safely changed after a failure there
  (matches the same "can't override 200 once committed" principle, just
  triggered earlier). Also had to widen `server.go`'s `downloadClient` test
  interface to add `ResolveTicketRange`/`DownloadManifestRange` (develop's
  `serveRange`/`serveHead` call them on `s.client`, which my interface
  narrowing didn't know about yet) and update the `fakeDownloadClient` test
  double to implement them (stubbed, unused by the timeout/error-mapping
  tests). The user completed the merge commit (`c55993e`) themselves right
  after the handler.go resolution landed; the interface-widening follow-up was
  a small separate uncommitted fix on top (go build succeeded without it since
  it's a test-only gap - only go vet/go test caught it).
- **go-sdk changes were committed+pushed by the user directly** (commit `a9739e3
  feat(download): add timeout bounds to download operations`, on `origin/main` of
  `/home/tuan/work/depin-workspace/go-sdk` - confirmed via `git fetch` + comparing
  local/origin `main` SHAs), not by me - I never ran `git commit`/`push` in that
  repo. Finalized depin's pin to it same-session: dropped the local `../go-sdk`
  directory replace, re-pinned `require`/`replace aioz-depin/go-sdk` to
  `gitlab.internal/aioz-depin/go-sdk v0.0.0-20260722153257-a9739e31ca2a` (a real
  pseudo-version, computed via a scratch module - see the updated recipe in
  [[edgeserver-metadata-headers]]), `go mod tidy`. `go build`/`go test` verified
  clean against the real pinned version (edgeserver + cmd/edgeserver + full
  module except the 3 already-pre-existing-broken spots: `cmd/uplink`'s broken
  imports, `internal/testplanet`'s `CreateContractWithPlacement` drift, and
  `worker/pieces_test`'s `trustpkg.Dialer` nil-arg issue - none touched by this
   version bump, all present before it too). Removed the now-unused local
   `../go-sdk` symlink.

## Fix v3: timeout now measures inactivity (2026-08-03)

The absolute whole-download deadline incorrectly aborted healthy large transfers
that continued making progress for more than two minutes. `WithDownloadTimeout`
now means maximum download inactivity instead:

- `DownloadManifest` and `DownloadManifestRange` create a sliding timer rather
  than `context.WithTimeout`.
- Every successful worker-piece read and every successful plaintext destination
  write resets the timer. Tracking piece reads is essential because a large first
  segment may be actively transferring before any plaintext can be emitted.
- Expiry cancels with `context.DeadlineExceeded` via `context.WithCancelCause`;
  `classifyDownloadErr` checks `context.Cause` so teardown symptoms remain mapped
  to a timeout for edge HTTP error handling.
- Edge keeps its five-minute configured value, now as a stall interval rather
  than a wall-clock request limit.
- RED/GREEN tests prove progress can continue beyond twice the configured
  interval while no progress expires. Repeated race tests, SDK root plus
  segment/piece packages, edge race tests, `git diff --check`, and a real
  `cmd/edgeserver` build pass.
