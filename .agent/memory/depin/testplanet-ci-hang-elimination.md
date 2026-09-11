---
type: fact
tags: [ci, testplanet, debugging, gitlab, race, shutdown, depin]
created: 2026-08-10
agent: main
---

"testplanet hangs in the post-merge CI pipeline, fine on develop, fine locally"
on `feat/implement-testcases` (fcfd1e0) / `feat/reward-flows` (7bc1331).

## ROOT CAUSE - FIXED 2026-08-10 (uncommitted)

`LibP2PListener` in `internal/grpcutil/connector/p2p/conn.go`. `Close()` did
`close(l.connChan)`, and `Accept()` selected over `<-l.connChan` and
`<-l.ctx.Done()`. After Close **both cases are ready and Go picks at random**;
the channel case yields a nil conn and returned `(nil, nil)` - success, no error.

grpc's `Serve` reads that as a successful accept and loops to call `Accept`
again, forever. So `Serve` never returns -> `muxGroup.Wait()` never returns ->
`Server.Run` never returns -> testplanet pays `closablePeer.Close`'s 5s
(`planet.go:217`) **plus** `planet.run.Wait()`'s 5s (`planet.go:463`) = the 10.2s
stall. Random select is exactly why it was intermittent and load-sensitive.

This also explains why `c8f831d` looked like a fix: it corrected the ordering of
listener-close vs server-stop, which made shutdown reach this path less often.
The path itself stayed broken.

Two further latent process-killers in the same type: `close(connChan)` was not
idempotent (`close of closed channel`; grpc calls Close from BOTH `GracefulStop`
and `Stop`), and the stream handler sends on that channel (`send on closed
channel`). Neither is recoverable.

**Fix:** never close `connChan` (cancel the ctx instead - safe to repeat, and it
stops both sides); check ctx *before* the channel in Accept; never return a nil
conn with a nil error - return an error wrapping `net.ErrClosed`; guard
`closeListener` with `sync.Once`; drain+close queued conns; have the stream
handler check `ctx.Err()` before sending.

**Verified:** new `listener_test.go` (7 tests) - 6 of 7 failed pre-fix; all pass
under `-race`. End to end: shutdown guard **0/12** failed (was 1/8), full package
**0/6** failed with **0 races and 0 stall warnings across 18 runs** (was 2/6).
Package time unchanged (242-262s).

**Test-harness trap:** my first probe helper only called `cancel()` without
closing the channel, so it exercised the one path that was never broken and
reported two false passes. When probing a concurrency bug, make the helper
reproduce the *real* state change.

## How it presented (the investigation)

**The shutdown deadlock recorded as fixed in [[peer-shutdown-deadlock]] (`c8f831d`)
is NOT fully fixed.** It became intermittent instead of universal, and the harness
silently absorbs it.

Reproduced locally: `TestPlanetShutdownDoesNotWaitOnUndrainedPeers` fails ~1 in 8
full-package runs at 4 CPUs under `-race`:

```
WARN  some peers did not stop in time; proceeding with shutdown
Error:    "10.210197056s" is not less than "3s"
Messages: planet took 10.210197056s to shut down; peers are not stopping on Close
```

10.2s is two stacked 5s windows, both in `internal/testplanet/planet.go`:
1. `closablePeer.Close` (line 217) waits `peer.Close()` + `<-peer.runFinished`, 5s cap
2. `planet.run.Wait()` (line 463), 5s cap

Both elapse whenever a peer's `Run` never returns. `planet.go:218` then sets
`peer.err = nil`, so **the stall is swallowed** - invisible except via that one
guard test. The harness comment at `planet.go:198-201` admits the cause outright:
"their Run/Close design never cleanly stops Serve".

**Why it only breaks CI:** stall frequency is scheduling-dependent. Rare on a
64-core workstation, common on a contended runner. At 10s per affected planet
across ~72 planets that is hundreds of seconds of pure teardown, and the first
ceiling it meets is the coverage job's *implicit* 600s (see below).

## Timeout arithmetic

| job | testplanet base | ceiling | source |
|---|---|---|---|
| `test` (`-race`) | 440.8s | 1800s | explicit `-timeout 30m` |
| `coverage` (`-coverpkg`) | 162.0s | **600s** | Go default - `scripts/coverage.sh:45` sets NO `-timeout` |

`develop` has no coverage job at all, which is why develop never showed it.

## Bugs found and their status

1. **`coord/gc/bloomfilter/storage.go` data race - FIXED (uncommitted).**
   `MemStorage.latest *Bundle` written by the GC observer (ranged-loop cycle,
   `Observer.Finish`) and read by the GC sender (`sender.sendLatest`), two
   long-lived goroutines under the same lifecycle group, no synchronisation.
   Failed `TestGCReclaimsDeletedPieces` in 2/6 runs. Added `sync.RWMutex`;
   verified **0 races in 8 runs**. Note `Latest` returns a shallow copy sharing
   the `Filters` map - safe only because bundles are never mutated after Upload.
2. **Residual shutdown stall - NOT fixed.** The above. This is the real CI cause.
3. **`scripts/coverage.sh:45` has no `-timeout`** - the implicit 600s ceiling.
   Not the cause, but the surface the stall breaches first.
4. **`coord/server/server.go:346`** - the p2p `Serve` is the only unwrapped
   `Serve` in the repo (361/373 use `ignoreShutdownErr`; `worker/server` wraps all
   three at 359/372/385). Returns `grpc.ErrServerStopped` at teardown. This is the
   `planet shutdown failed: server closed` flake from [[repair-overhaul]], fixed in
   worker/server and missed on coord's p2p path.

## Measurement traps that cost me hours

- **Measuring a package in isolation measures the wrong thing.** testplanet alone
  under `-coverpkg` = 115s; in the real `./...` run = 440s (4x), because it then
  contends with `coord/db` etc. for the same Postgres. I ruled out the coverage
  job on the 115s number and was wrong.
- **`-race` accounts for the 440s vs 162s gap**, not coverage instrumentation
  (~4%). Don't compare a race build against a non-race one.
- **testplanet is latency-bound, not CPU-bound.** Pinning to 4 CPUs and running
  two suites concurrently changed the time by ~1%. "The runner is smaller" is a
  weak hypothesis here.
- **A single green run proves nothing against a flake.** Run 6-8 iterations and
  compare rates.

## Eliminated with measurements (don't re-test)

CoinGecko/network egress (all testplanet tests inject `fixedRate`/`fakeSender`/
`RateSourceFixed`); coverage instrumentation cost; goroutine/memory leak (RSS flat
0.9-1.3 GB, 12-13 threads); GitLab merged-results divergence (`rev-list --count
<branch>..origin/develop` = 0 for both branches); CPU starvation; test+coverage
stage contention.

## Environment

Workstation 64 cores / 125 GB - most of why "it runs fine locally" proves little.
Local test Postgres `depin-citest-pg` already matches CI (`max_connections=500`,
`shared_buffers=256MB`):
`DEPIN_TEST_POSTGRES='postgresql://depin:depin@localhost:5599/depin_test?sslmode=disable'`

Related: [[peer-shutdown-deadlock]], [[repair-overhaul]], [[testplanet-coverage-plan]].
