---
type: decision
tags: [shutdown, grpc, serveWG, http2-preface, testplanet, flaky-test, coord, worker, conn-leak]
created: 2026-08-12
agent: main
---

**Fixed the flaky `TestPlanetShutdownDoesNotWaitOnUndrainedPeers` on branch
`feat/implement-testcases`.** Uncommitted (no-auto-commit rule). This is a **third**
distinct cause of the same 10.2s testplanet stall - after the Close/Run ordering
deadlock and after [[testplanet-ci-hang-elimination]]'s `LibP2PListener.Accept`
returning `(nil, nil)`. Same symptom, different mechanism each time, so treat the
10.2s signature as a class of bug rather than one fixed bug.

## Symptom
CI (and local, ~1 in 5 runs with `-race`) fails with
`"10.20524846s" is not less than "3s"` plus
`WARN some peers did not stop in time; proceeding with shutdown`.
10.2s is exactly the two stacked 5s bounds in `internal/testplanet/planet.go`:
`closablePeer.Close` (line 217) then `planet.run.Wait()` (line 463). Both elapsing
in full means a genuine hang, not slowness - useful discriminator, do not chase
"CI is loaded" for this shape.

## Root cause (proven from a goroutine dump, not inferred)
The **coordinator's public TCP grpc server** is the stuck peer (not a worker):

1. 4 goroutines parked in
   `grpc.Serve.func3 -> handleRawConn -> newHTTP2Transport -> NewServerTransport ->
   io.ReadFull(24 bytes) -> tls.Conn.Read`.
   **24 bytes = the HTTP/2 client preface.** TLS had already completed; the client
   connected, handshook, and then never started an HTTP/2 transport.
2. grpc's accept loop does `s.serveWG.Add(1)` **per accepted conn** for the whole
   duration of the HTTP/2 handshake (`grpc@v1.74.2/server.go:955-960`).
3. `stop()` (shared by both `Stop` and `GracefulStop`) does `s.serveWG.Wait()` at
   `server.go:1947`, **before** `defer s.done.Fire()` and before it touches any
   transport. So it blocks.
4. `Serve`'s own defer (`server.go:882-889`) does `serveWG.Done()` then, because
   `s.quit.HasFired()`, blocks on `<-s.done.Done()` - which step 3 never reaches.
   Serve never returns.
5. -> `muxGroup.Wait()` in `coord/server.Server.Run` never returns -> `p.wg.Wait()`
   in `Server.Close` blocks -> both testplanet bounds elapse.

`grpcutil.StopServer`'s forced `go s.Stop()` escape hatch **cannot** help: Stop
funnels into the same `stop()` and the same `serveWG.Wait()`. Its doc comment
already suspected this; the comment's "idle client parked in ReadFrame" is actually
imprecise - an *established* idle conn is held in `s.conns`, which `Stop` does
handle. Only a **pre-preface** conn pins `serveWG`.

## Why a client goes silent mid-handshake (our own doing)
`connector.TCPConnector.DialContext` completes TCP **and the TLS handshake**
eagerly, then `grpcconn.New` calls `grpc.NewClient`, which is **lazy** - it writes
the preface only when an RPC needs the transport. So any dialed-but-never-used
connection is, to the server, a client that connected and went silent. grpc bounds
it only by `connectionTimeout` (default **120s**), set via
`rawConn.SetDeadline` in `handleRawConn`.

That also explains the nondeterminism: see the client-side leak below - the sockets
are reclaimed only by a **GC finalizer**, so whether GC ran before teardown decides
pass/fail.

## The fix (server-side, bounded regardless of clients)
New `internal/grpcutil/conntrack.go`: `TrackedListener` wraps a `net.Listener`,
records every accepted conn (`trackedConn` deregisters itself on Close via
`sync.Once`), and exposes `CloseConns()` / `ConnCount()`.

Wired into **both** `coord/server` and `worker/server`:
- `trackConns()` wraps every listener (public TCP, private TCP, and the p2p
  listener created inside `Run`) and registers the wrapper.
- `Close()` calls `p.closeTrackedConns()` **after** `stopGRPCServer(...)` (so the
  bounded graceful window is preserved) and **before** `p.wg.Wait()`.
- Trackers are guarded by a **separate `trackerMu`, not `p.mu`**: `Close` holds
  `p.mu` for its whole body including the wait on `Run`, while `Run` registers the
  p2p tracker - sharing the mutex deadlocks. Register-then-check-`p.done` ordering
  makes the Close-ran-first race safe.
- `trackedConn` exposes `NetConn()` because `connector.isLimited` walks the conn
  chain through that conventional accessor; a wrapper without it truncates the walk.

Also fixes production: SIGINT drain was bounded only by grpc's 120s, which the
worker-updater's binary swap relies on.

## The client-side half - ALSO FIXED (this was the actual source)
Three omissions stacked so that **no caller could ever release a dialed connection**:

1. `internal/grpcutil/dial/conn.go` - `func (c *Conn) Close() error { return nil }`.
2. It could not be anything else: `dial.Conn` **embedded** `io.Closer` and
   `dialPool` never set it, so a real call would panic on the nil interface. The
   no-op was papering over that panic.
3. And it could not be set: **`pool.Conn`, the interface `Pool.Get` returns,
   declared no `Close`** - only `ClientConnInterface` + `State()`. The concrete
   `*poolConn` has a correct `Close`; the interface hid it.
4. On top of that, 7 of 10 call sites returned `grpc.ClientConnInterface` from
   helpers (`dialFetcher.dialWorker`, `Usecase.dialCoord`, ...), which also has no
   `Close`.

Fix: added `io.Closer` to `pool.Conn`; replaced `dial.Conn`'s embedded closer with a
named nil-tolerant `closer` field wired to the pooled handle; changed the four
helpers to return `*dial.Conn`; added `defer conn.Close()` at all ten sites
(`coord/audit` x4, `coord/contact`, `coord/gc/sender`, `coord/repair/repairer`,
`worker/contact` x2, plus the three that already "closed" into the no-op:
`worker/orders`, `worker/relay`, `worker/trust`).

**`worker/trust/pool.go` was the actual leak that stalled the coordinator**: it
force-dials the coord every refresh, reads the peer identity off the TLS state,
never issues an RPC, and "closed" into the no-op. That is exactly a completed-TLS,
no-preface socket. 4 workers -> the 4 stalled conns in the dump, and the failure log
shows all four running `trust ... Refresh` in the same millisecond as shutdown.

Checked before changing behaviour: every stream is fully drained inside its dialing
function (`Download`, `CloseAndRecv`), so no `defer Close()` kills a live stream;
`segmentverify`'s shared pool means `ownsPool == false` there so conns stay cached
(no reuse regression); `Cache.Put` on a closed cache closes the value rather than
storing it; the pool's close callback defers to `release()` for limited conns with
`inFlight > 0`. **`poolConn.Close()` ends in `return nil`, discarding the pool's
error** - a real wart, left alone deliberately so a cache-close error cannot fail an
already-successful settlement RPC.

## Verification
- Baseline flake: 1/5 then 1/7 runs, 15.1s.
- **Causal proof**: with the server-side fix *disabled* and only the client fix in
  place, 20/20 green. The unclosed dials were the source, not merely a candidate.
- Both fixes: 20/20 green; full `-race` CI suite green (108 packages, exit 0).
- The two fixes are independent layers: the client fix removes the cause, the server
  fix bounds shutdown even against a genuinely misbehaving remote client.

## Regression test gotcha
`internal/grpcutil/conntrack_test.go` encodes the mechanism: a raw `net.Dial` that
sends nothing, `StopServer`, assert Serve is *still* blocked, then `CloseConns()` and
assert Serve returns. Two traps cost me two wrong versions of it:
- **Do NOT close the listener before `StopServer`.** grpc's `Serve` only parks on
  `<-s.done.Done()` when `quit` has already fired; close the listener first and Serve
  returns early with a plain accept error, proving nothing. Let grpc's own `stop()`
  close the listener - it fires quit first, so the ordering is deterministic.
- **Connection count is not a sufficient barrier.** It rises at `Accept`, but
  `handleRawConn` *starts* with its own `if s.quit.HasFired() { rawConn.Close() }`.
  Stop inside that window and the handler bails cleanly. Wait for the server's
  SETTINGS frame instead (grpc writes+flushes it at http2_server.go:208/300,
  immediately before the preface read) - one byte on the client proves the handler is
  parked in the right read.
Mutation-checked: neutering `CloseConns` makes it fail after 10.3s.

## Technique worth reusing
To find a shutdown hang, copy the failing test into a throwaway
`zz_debug_*_test.go` that calls `pprof.Lookup("goroutine").WriteTo(os.Stderr, 2)`
when the budget is blown, then loop it until it fails. The abandoned goroutines are
still parked at that point, so the dump names the culprit. Low goroutine numbers in
the dump = created early in the run, which is what revealed that two of the four
stalled sockets had been sitting there since planet start.

Local test Postgres for this repo:
`docker run -d --rm --name depin-pgtest -e POSTGRES_USER=depin -e POSTGRES_PASSWORD=depin -e POSTGRES_DB=depin_test -p 15445:5432 postgres:15 -c max_connections=500`
then `CREATE EXTENSION IF NOT EXISTS "uuid-ossp";` up front (migration 001's
`IF NOT EXISTS` is not concurrency-safe across parallel packages).
