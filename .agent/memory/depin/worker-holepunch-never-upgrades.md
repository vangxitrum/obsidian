---
type: fact
tags: [depin, worker, relay, libp2p, nat, holepunch, dcutr, throughput]
created: 2026-08-03
agent: main
---

**Why fleet workers never get a direct (hole-punched) connection: they are
Docker containers with no published port, behind double NAT with source-port
rewriting. Hole punching cannot work for them as deployed.**

First, a premise correction that matters: a worker DOES have a direct
connection to the relay - `p2pmonitor` reports `direct_peers: 1`, which is the
relay itself. Workers dial the relay directly (that is how the reservation is
made); the relay is public. DCUtR/hole punching is not worker->relay. What is
actually stuck at zero is `direct_upgrades_total`: relayed **worker <-> client
/ edge** connections never get upgraded to direct, so every piece transfer
keeps paying the relay hairpin.

Verified facts (2026-08-03, fleet on this workstation, relay 68.183.189.51):

- Worker libp2p listen addr: `/ip4/172.29.0.30/tcp/7777` - Docker bridge
  (`depin-fleet`, driver bridge, subnet 172.29.0.0/16), i.e. a PRIVATE address.
- `docker port fleet-worker-1` is **empty**. The Dockerfile only EXPOSEs
  7777/tcp; nothing is published to the host. So there is no inbound path from
  the host's public interface to the container's listen port at all.
- Relay logs show workers arriving from `/ip4/118.69.133.193/tcp/37277` - a
  public IP but an **ephemeral source port**, not 7777. That is Docker bridge
  masquerade (plus the office NAT) rewriting the source port: effectively
  symmetric NAT.

Why libp2p then cannot help (go-libp2p v0.46.0):

- `holepunch.Service.waitForPublicAddr()` (p2p/protocol/holepunch/svc.go:124)
  does not even register the DCUtR stream handler until `HolePunchAddrs()`
  returns something.
- `addrsManager.HolePunchAddrs()` (p2p/host/basic/addrs_manager.go) =
  DirectAddrs + observed addrs, then
  `DeleteFunc(!manet.IsPublicAddr)`. The worker's direct addr is 172.29.x, so it
  is dropped; only an observed address can qualify.
- The observed address that survives is `118.69.133.193:<ephemeral>`, which is a
  per-flow SNAT binding toward the RELAY. Any other peer dialing it hits a port
  that is closed or bound to a different flow. Classic symmetric-NAT hole-punch
  failure.
- `holePuncher.directConnect` (holepuncher.go:108) only short-circuits to a
  direct dial when the REMOTE has a public non-relay address; that path can work
  edge->worker only if the worker is reachable, which it is not.
- Transport is TCP only (no QUIC/UDP listener). TCP simultaneous-open hole
  punching is much weaker than QUIC's and effectively requires port-preserving
  NAT.

`worker/server/server.go` already sets `EnableAutoNATv2`, `EnableRelay`,
`EnableHolePunching`, and deliberately `ForceReachabilityPrivate()` when static
relays are configured. That configuration is correct and is NOT the bug - the
worker genuinely is unreachable, so forcing private is honest.

Fix directions (none is a code bug; all are deployment/topology):
1. Publish the worker p2p port to the host and forward it at the router, so the
   worker has a real public addr - then no hole punch is needed at all.
   For the dev fleet, `-p <hostport>:7777` per container plus announcing that
   address. Cheapest correct fix for a lab.
2. Add a QUIC/UDP listener; UDP hole punching succeeds far more often than TCP
   and is what DCUtR is designed around.
3. Use host networking for fleet containers to remove one NAT layer (still
   leaves the office NAT).
4. Accept relay-only and scale the relay - this is the current de-facto state,
   and is why relay throughput (~13.9-15.4 MB/s observed) is the ceiling for
   aggregate download speed. See [[download-benchmark-harness]].

Related: [[relay-domain-locator-egress]] (the dual-address / locator work),
[[relay-connected-peers-vs-reservations]].

## ACTUAL root cause (proved 2026-08-03 with libp2p debug logs)

The NAT analysis above is real but is NOT the blocker. Ran one worker with
`GOLOG_LOG_LEVEL='error,p2p-holepunch=debug,...'` (throwaway container `wdebug`
reusing worker-1's identity while worker-1 was stopped; fleet restored after).
Sequence observed:

1. `p2p-holepunch` DOES activate on the worker: "Host now has a public address",
   addresses `[/ip4/127.0.0.1/tcp/7777 /ip4/172.29.0.30/tcp/7777 /dns/relay/...
   /p2p-circuit]`. (Note it counted 127.0.0.1 as qualifying - the gate is
   weaker than expected.)
2. On each inbound relayed conn the worker DOES call `beginDirectConnect` and
   "attempting direct dial".
3. **`addrs=[]`** - the worker has NO dialable address for the remote peer, so
   the direct-dial short-circuit (holepuncher.go:117) can never fire.
4. Hole punch then fails immediately:
   `failed to open hole-punching stream: failed to negotiate protocol:
   protocols not supported: [/libp2p/dcutr]`

**The remote peer does not speak /libp2p/dcutr.** The peer is
`QmTZwkE99FvgXBnW8gwxGZZzSwXJf1ybu1qwVJ3FVc188A`, which calls
`ContactService/Ping` over the relay circuit - i.e. coord.

Both `coord/server/server.go:169-177` and
`internal/grpcutil/connector/p2p_connector.go:97-105` DO pass
`libp2p.EnableHolePunching()`. But holepunch registers its stream handler only
inside `waitForPublicAddr` (svc.go:124-141) - `SetStreamHandler(Protocol, ...)`
runs only once `HolePunchAddrs()` is non-empty. If coord's host never sees a
qualifying address, the DCUtR protocol is never advertised, and every worker's
hole punch dies at negotiation. That is consistent with `addrs=[]`: the worker
learns no dialable address for coord via identify either.

So hole punching is broken on the **coord/edge side**, not the worker side, and
would stay broken with host networking or a perfect NAT. Next step is to run
coord (and the edge) with `GOLOG_LOG_LEVEL=p2p-holepunch=debug` and check
whether they log "Host now has a public address" or stay stuck on "waiting
until we have at least one public address".

Do NOT migrate the fleet to host networking as a hole-punching fix until the
coord/edge side advertises /libp2p/dcutr and a dialable address.

## Config-driven libp2p logging (2026-08-03)

Diagnosing this needed `GOLOG_LOG_LEVEL`, which is awkward to deploy. Made it a
config-file setting instead: **`log.libp2p-level`** in config.yaml, e.g.

```yaml
log.libp2p-level: "error,p2p-holepunch=debug"
```

Empty (default) leaves libp2p on its own GOLOG_LOG_LEVEL behavior.

Why it needed care: go-libp2p reads `GOLOG_LOG_LEVEL` **once at package init**,
long before main() parses a config file. Proved with a throwaway program that
`os.Setenv` in main() has no effect (`debug enabled: false`).

The way through is `gologshim.SetDefaultHandler(slog.Handler)`. Each libp2p
subsystem defers picking its handler until its FIRST log call
(`dynamicHandler.ensureHandler`, gologshim.go:80) and takes `defaultHandler` if
set - explicitly "to handle init order issues". `Enabled()` also delegates to
that handler, so our handler controls level too, fully bypassing the env.
Verified with a second throwaway program: handler installed in main(), no
GOLOG_LOG_LEVEL at all -> debug records flow.

Implementation: `pkg/process/libp2plog.go` - a `slog.Handler` forwarding into
zap (no new dependency; go-log/v2 is NOT in the module graph, and is not needed
since SetDefaultHandler takes any slog.Handler). gologshim tags each subsystem
via `WithAttrs(logger=<system>)` before any Enabled/Handle, so the handler
captures the name there to resolve its per-subsystem level.

**Hook point matters.** First attempt put it in `process.NewLogger` - wrong:
binaries call that in main() BEFORE `process.Exec*` reads the config, so the
flag was still "". Correct location is `pkg/process/exec_conf.go`, right after
"Configuration loaded" and before any command body (hence before any libp2p
host). Symptom of getting it wrong: config key present, zero libp2p log lines.

Parsing gotcha: `zapcore.Level.UnmarshalText("")` returns InfoLevel WITHOUT an
error, so a bare "warn" (where strings.Cut puts the text in `name`, not
`levelText`) silently became info. Guard the empty string explicitly.

Bonus: because libp2p now logs through zap, these lines also reach Loki via
logship - so p2p problems on remote coord/edge deployments become diagnosable
without shell access. That is the intended next step here: set
`log.libp2p-level: "error,p2p-holepunch=debug"` on coord and the edge and check
whether they ever log "Host now has a public address", which decides whether
they advertise /libp2p/dcutr at all.
