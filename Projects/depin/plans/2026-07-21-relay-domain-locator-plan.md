# Relay Domain Locator + Split-Horizon DNS — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Put a DNS name (not a raw IP) in the worker's relay circuit locator so our in-VPC edge/coordinator resolve the relay to its private IP (free VPC transfer) while external SDK clients resolve to the public IP, cutting the billed `relay → edge` egress.

**Architecture:** The worker composes its external contact locator in `worker/relay/reserve.go` from the operator-configured relay addr; that string is stored by the coordinator and signed into download order limits. We (1) set the configured relay addr to a `/dns4/` name and (2) make the worker preserve that name verbatim in the locator even when AutoRelay has observed the relay's IP. Split-horizon authoritative DNS then hands each dialer the right IP. libp2p resolves `/dns4/` at dial time, so no dial-path code changes.

**Tech Stack:** Go, go-libp2p (AutoRelay / circuit-relay v2), go-multiaddr, testify, zap/zaptest.

## Global Constraints

- **Locator form:** use `/dns4/<host>` only — never `/dns6/` or raw `/ip6/`. The `internal/vo/peer_url.go` parser splits on `:`; `/dns4/` is colon-free, IPv6 forms are not.
- **DNS safety rule:** the relay's private A record must NEVER appear in the external DNS view. A leak makes external downloads fail.
- **Storj alignment:** keep components aligned with Storj (`../storj`) where a counterpart exists (project rule, `CLAUDE.md`).
- **Commits:** do NOT run `git commit`/push without explicit user go-ahead (project rule). The commit steps below mark the intended commit boundaries; treat them as checkpoints to *request* approval, not license to auto-commit.
- **Branch:** work on `feat/relay-domain-locator` (current branch is `develop`; do not commit onto it directly).
- **Domain string:** the relay hostname is a deployment parameter. This plan uses `relay.example.com` as a stand-in — substitute the real hostname at config time; no code depends on the specific string.

---

## File Structure

- **`worker/relay/reserve.go`** (modify) — the only production code change. Add three helpers (`relayIDFromCircuit`, `findConfiguredRelay`, `selectAdvertisedLocator`) and route `WaitForRelayReservation` through them so the reported locator carries the configured relay addr (domain-preserving) rather than AutoRelay's observed IP.
- **`worker/relay/reserve_test.go`** (create) — unit tests for the pure locator-selection logic and the relay-ID extractor.
- **`coord/contact/service.go`** (config value only, no code) — `RelayAddrs` is where the operator sets the `/dns4/` relay addr distributed to workers.
- **`dev/workers/p2p/worker-*/config.yaml`** (config value only) — dev/standalone workers' static `worker.relay-addrs` override, updated for local end-to-end testing.
- **Ops (no repo code):** authoritative split-horizon DNS views; relay listen check; relay public-egress monitor.

---

### Task 1: Preserve the configured relay domain in the worker locator

**Files:**
- Modify: `worker/relay/reserve.go` (add helpers near `advertisedCircuitAddr` at :81; change the call site in `WaitForRelayReservation` at :122)
- Test: `worker/relay/reserve_test.go` (create)

**Interfaces:**
- Consumes: existing `isCircuitAddr(ma.Multiaddr) bool`, `externalAddr(libp2ppeer.ID, ma.Multiaddr) string`, `constructedCircuitAddr(libp2ppeer.ID, libp2ppeer.AddrInfo) (string, bool)` (all already in `reserve.go`).
- Produces:
  - `selectAdvertisedLocator(hostID libp2ppeer.ID, hostAddrs []ma.Multiaddr, relays []libp2ppeer.AddrInfo) (string, bool)`
  - `relayIDFromCircuit(a ma.Multiaddr) (libp2ppeer.ID, bool)`
  - `findConfiguredRelay(relays []libp2ppeer.AddrInfo, id libp2ppeer.ID) (libp2ppeer.AddrInfo, bool)`

- [ ] **Step 1: Write the failing tests**

Create `worker/relay/reserve_test.go`:

```go
package relay

import (
	"testing"

	libp2ppeer "github.com/libp2p/go-libp2p/core/peer"
	ma "github.com/multiformats/go-multiaddr"
	"github.com/stretchr/testify/require"
)

// Two real, decodable peer IDs (relay + worker) reused across the locator tests.
const (
	testRelayIDStr  = "QmdVWkhNSV7SbprCNqgM9xCJ7bGuswKpeWXXYyvBU5gQSq"
	testWorkerIDStr = "QmSRUwE3DSAy7yN8u7pBnRuBc5WJw58mAGtwBeznc42TNU"
)

func mustDecodeID(t *testing.T, s string) libp2ppeer.ID {
	t.Helper()
	id, err := libp2ppeer.Decode(s)
	require.NoError(t, err)
	return id
}

// advertised circuit addr as AutoRelay would surface it: relay reached by IP.
func advertisedIPCircuit(t *testing.T, relayID libp2ppeer.ID) ma.Multiaddr {
	t.Helper()
	a, err := ma.NewMultiaddr(
		"/ip4/68.183.189.51/tcp/7781/p2p/" + relayID.String() + "/p2p-circuit",
	)
	require.NoError(t, err)
	return a
}

func TestSelectAdvertisedLocator_RewritesToConfiguredDomain(t *testing.T) {
	relayID := mustDecodeID(t, testRelayIDStr)
	workerID := mustDecodeID(t, testWorkerIDStr)

	hostAddrs := []ma.Multiaddr{advertisedIPCircuit(t, relayID)}
	configured := []libp2ppeer.AddrInfo{{
		ID:    relayID,
		Addrs: []ma.Multiaddr{ma.StringCast("/dns4/relay.example.com/tcp/7781")},
	}}

	got, ok := selectAdvertisedLocator(workerID, hostAddrs, configured)
	require.True(t, ok)
	require.Equal(t,
		"p2p:"+workerID.String()+
			":/dns4/relay.example.com/tcp/7781/p2p/"+relayID.String()+"/p2p-circuit",
		got,
	)
}

func TestSelectAdvertisedLocator_FallbackWhenRelayNotConfigured(t *testing.T) {
	relayID := mustDecodeID(t, testRelayIDStr)
	workerID := mustDecodeID(t, testWorkerIDStr)

	hostAddrs := []ma.Multiaddr{advertisedIPCircuit(t, relayID)}
	// Configured set does NOT contain the advertised relay -> use advertised as-is.
	got, ok := selectAdvertisedLocator(workerID, hostAddrs, nil)
	require.True(t, ok)
	require.Equal(t,
		"p2p:"+workerID.String()+
			":/ip4/68.183.189.51/tcp/7781/p2p/"+relayID.String()+"/p2p-circuit",
		got,
	)
}

func TestSelectAdvertisedLocator_NoCircuitAddr(t *testing.T) {
	workerID := mustDecodeID(t, testWorkerIDStr)
	hostAddrs := []ma.Multiaddr{ma.StringCast("/ip4/10.0.0.5/tcp/7777")}
	_, ok := selectAdvertisedLocator(workerID, hostAddrs, nil)
	require.False(t, ok)
}

func TestRelayIDFromCircuit(t *testing.T) {
	relayID := mustDecodeID(t, testRelayIDStr)
	got, ok := relayIDFromCircuit(advertisedIPCircuit(t, relayID))
	require.True(t, ok)
	require.Equal(t, relayID, got)
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `go test ./worker/relay/ -run 'SelectAdvertisedLocator|RelayIDFromCircuit' -v`
Expected: FAIL — compile error `undefined: selectAdvertisedLocator` / `undefined: relayIDFromCircuit`.

- [ ] **Step 3: Add the helpers and rewrite the advertised-locator selection in `reserve.go`**

In `worker/relay/reserve.go`, replace the existing `advertisedCircuitAddr` function (currently at :81-88) with the domain-preserving version plus its two helpers:

```go
// selectAdvertisedLocator returns the worker's external circuit locator once
// AutoRelay advertises a live circuit reservation. It treats the advertisement only
// as the "reservation is live" signal, then rewrites the relay component of the
// locator to the operator-configured relay addr (matched by relay peer ID) so a
// configured DNS name (e.g. /dns4/relay.example.com/...) survives verbatim instead of
// being replaced by whatever IP AutoRelay observed while dialing the relay. If the
// advertised relay is not in the configured set, the advertised address is used as-is.
func selectAdvertisedLocator(
	hostID libp2ppeer.ID,
	hostAddrs []ma.Multiaddr,
	relays []libp2ppeer.AddrInfo,
) (string, bool) {
	for _, a := range hostAddrs {
		if !isCircuitAddr(a) {
			continue
		}
		if relayID, ok := relayIDFromCircuit(a); ok {
			if cfg, found := findConfiguredRelay(relays, relayID); found {
				if ext, ok := constructedCircuitAddr(hostID, cfg); ok {
					return ext, true
				}
			}
		}
		// Advertised relay not in the configured set (unexpected) — use as-is.
		return externalAddr(hostID, a), true
	}
	return "", false
}

// relayIDFromCircuit extracts the relay's peer ID from an advertised circuit
// multiaddr of the form <relay-addr>/p2p/<relay-id>/p2p-circuit.
func relayIDFromCircuit(a ma.Multiaddr) (libp2ppeer.ID, bool) {
	base := a.Decapsulate(ma.StringCast("/p2p-circuit"))
	if base == nil {
		return "", false
	}
	info, err := libp2ppeer.AddrInfoFromP2pAddr(base)
	if err != nil {
		return "", false
	}
	return info.ID, true
}

// findConfiguredRelay returns the configured relay AddrInfo whose peer ID matches id.
func findConfiguredRelay(
	relays []libp2ppeer.AddrInfo,
	id libp2ppeer.ID,
) (libp2ppeer.AddrInfo, bool) {
	for _, r := range relays {
		if r.ID == id {
			return r, true
		}
	}
	return libp2ppeer.AddrInfo{}, false
}
```

- [ ] **Step 4: Route `WaitForRelayReservation` through the new selector**

In `worker/relay/reserve.go`, inside `WaitForRelayReservation`, change the preferred-path check (currently `reserve.go:122`) from:

```go
		// Preferred: AutoRelay advertised a circuit addr (routable relay).
		if ext, ok := advertisedCircuitAddr(host); ok {
			return ext, nil
		}
```

to:

```go
		// Preferred: AutoRelay advertised a live circuit; report it with the relay
		// component rewritten to the configured addr so a /dns4/ name is preserved.
		if ext, ok := selectAdvertisedLocator(host.ID(), host.Addrs(), relays); ok {
			return ext, nil
		}
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `go test ./worker/relay/ -run 'SelectAdvertisedLocator|RelayIDFromCircuit' -v`
Expected: PASS (4 tests).

- [ ] **Step 6: Build and vet the worker to catch wiring regressions**

Run: `go build ./worker/... && go vet ./worker/relay/`
Expected: no output (success).

- [ ] **Step 7: Commit (request approval first — see Global Constraints)**

```bash
git checkout -b feat/relay-domain-locator   # only if not already on it
git add worker/relay/reserve.go worker/relay/reserve_test.go
git commit -m "feat(worker/relay): preserve configured relay domain in circuit locator"
```

---

### Task 2: Set the relay locator to a /dns4/ name (config)

**Files:**
- Modify (value only): the operator config that sets `coord/contact/service.go` `RelayAddrs` (the production knob distributed to workers via `GetRelayAddrs`).
- Modify (value only): `dev/workers/p2p/worker-*/config.yaml` — `worker.relay-addrs` for local end-to-end testing.

**Interfaces:**
- Consumes: `selectAdvertisedLocator` from Task 1 (guarantees the configured addr's domain reaches the locator).
- Produces: worker-reported locators of the form `p2p:<worker>:/dns4/<host>/tcp/7781/p2p/<relay-id>/p2p-circuit`.

- [ ] **Step 1: Point the dev worker configs at the /dns4/ relay addr**

For each `dev/workers/p2p/worker-*/config.yaml`, change the relay addr from the IP form to the domain form (substitute the real hostname for `relay.example.com`):

```yaml
worker.relay-addrs: /dns4/relay.example.com/tcp/7781/p2p/QmdVWkhNSV7SbprCNqgM9xCJ7bGuswKpeWXXYyvBU5gQSq
```

Bulk edit (from repo root), then spot-check one file:

```bash
grep -rl 'worker.relay-addrs: /ip4/68.183.189.51/tcp/7781/p2p/' dev/workers/p2p/ \
  | xargs sed -i 's#worker.relay-addrs: /ip4/68.183.189.51/tcp/7781/p2p/#worker.relay-addrs: /dns4/relay.example.com/tcp/7781/p2p/#'
grep -m1 'worker.relay-addrs' dev/workers/p2p/worker-2/config.yaml
```
Expected: prints `worker.relay-addrs: /dns4/relay.example.com/tcp/7781/p2p/QmdVWkhNSV7SbprCNqgM9xCJ7bGuswKpeWXXYyvBU5gQSq`.

- [ ] **Step 2: Set the production `RelayAddrs` value**

The operator sets coord `RelayAddrs` (help text at `coord/contact/service.go:49`) via the coordinator's config/env to the same `/dns4/` form. This is a deployment config value, not a code edit. Record the exact value in the deploy config (e.g. `coord` config file / `AIOZ_*` env) as:

```
/dns4/relay.example.com/tcp/7781/p2p/QmdVWkhNSV7SbprCNqgM9xCJ7bGuswKpeWXXYyvBU5gQSq
```

- [ ] **Step 3: Verify the config parses (no code path rejects /dns4/)**

Run a quick parse check that the worker's relay parser accepts the domain form:

```bash
go test ./worker/relay/ -run 'ParseRelayAddr' -v
```
Expected: PASS. If no such test exists, add one asserting `ParseRelayAddrInfos([]string{"/dns4/relay.example.com/tcp/7781/p2p/QmdVWkhNSV7SbprCNqgM9xCJ7bGuswKpeWXXYyvBU5gQSq"}, log)` returns exactly one AddrInfo. (Reuse the `zaptest.NewLogger(t)` pattern from `bootstrap_test.go`.)

- [ ] **Step 4: Commit (request approval first)**

```bash
git add dev/workers/p2p/
git commit -m "chore(dev): point dev worker relay-addrs at /dns4/ relay name"
```

---

### Task 3: Authoritative split-horizon DNS + relay listen + egress monitor (ops)

This task is operational (no repo code). It is required for the code + config changes to actually save bandwidth. Capture it in the deploy runbook.

**Files:**
- Ops/runbook only (no repo source). If the repo tracks infra, add the DNS zone/records and the monitor rule there.

- [ ] **Step 1: Confirm the relay listens on all interfaces**

On the relay host, confirm it accepts connections on the private interface:

```bash
ss -ltnp | grep ':7781'
```
Expected: bound to `0.0.0.0:7781` (or `*:7781`), not only the public IP. If bound to a single public address, change the relay's listen config to `0.0.0.0:7781` and restart.

- [ ] **Step 2: Create the two authoritative DNS views**

At the authoritative DNS for the relay hostname, create split-horizon views selected by source:
- **internal view** — match the VPC source ranges (the CIDRs of edge + coord + relay) → `A <relay-private-IP>`.
- **external view** — everything else → `A <relay-public-IP>`.

Hard rule (Global Constraints): the private A record must never appear in the external view.

- [ ] **Step 3: Verify resolution from each vantage point**

From an in-VPC host (edge or coord):
```bash
dig +short relay.example.com
```
Expected: the **private** IP.

From an external host (or public resolver):
```bash
dig +short relay.example.com @1.1.1.1
```
Expected: the **public** IP.

- [ ] **Step 4: Add a relay public-egress monitor**

Add an alert on the relay's **public** interface egress (or the relay egress metric) so that a misconfigured internal view — which silently keeps paying — is visible. Wire it into the existing monitoring stack (`monitoring/`), matching the relay dashboard conventions already in `monitoring/grafana/dashboards/relay/`.

- [ ] **Step 5: Record the operational steps in the deploy runbook**

Document Steps 1-4 (relay listen, the two views, the never-leak rule, the egress monitor) in the deploy runbook so the setup is reproducible.

---

### Task 4: End-to-end verification (real download path)

**Files:**
- No source changes — verification only.

- [ ] **Step 1: Bring up a worker with the /dns4/ relay addr and confirm the reported locator**

Start a worker (dev config from Task 2). Confirm the address the coordinator stores/echoes for that worker contains the domain, not the IP:

```
grep -i 'relay circuit address ready' <worker-log>     # worker side: external_addr=p2p:<worker>:/dns4/relay.example.com/...
```
Cross-check on the coordinator that the stored worker address / the `WorkerAddress` it signs into order limits also carries `/dns4/relay.example.com/...`. This proves Task 1 preserved the domain end-to-end into the order limit.

- [ ] **Step 2: Edge download stays on the private path**

From the edge box (in-VPC, resolves the domain to the private IP), run a real download through the edge/S3 API. While it runs, watch the relay's **public** interface counter:

```bash
# on the relay, before/after the download:
cat /sys/class/net/<public-iface>/statistics/tx_bytes
```
Expected: the public tx_bytes does NOT climb meaningfully for that transfer (traffic egressed on the private interface instead).

- [ ] **Step 3: External SDK download still works over the public path**

From an external client (resolves the domain to the public IP), run the same download via the go-sdk direct path. Expected: succeeds. This proves the split-horizon external view is intact.

- [ ] **Step 4: Coordinator dials the relay privately**

Trigger an audit or repair that dials a relayed worker from the coordinator (in-VPC). Confirm the coord→relay leg uses the private IP (e.g. relay connection log shows the coord's private-range source address). Bonus saving, and a regression check on the internal view.

---

## Self-Review

**1. Spec coverage:**
- Spec "Changes required" #1 (config /dns4/) → Task 2. ✅
- #2 (authoritative split-horizon DNS) → Task 3 Steps 2-3. ✅
- #3 (reserve.go domain-preserving locator) → Task 1. ✅
- #4 (peer_url.go no change; /dns4 colon-safe) → covered by Global Constraints + Task 2 Step 3 parse check. ✅
- #5 (relay listens 0.0.0.0) → Task 3 Step 1. ✅
- Verification plan (worker reports /dns4, edge private, sdk public, coord private) → Task 4 Steps 1-4. ✅
- Risk: misconfigured internal view silently pays → Task 3 Step 4 egress monitor. ✅
- Decision: Approach A only (C deferred) → no dual-address tasks present. ✅

**2. Placeholder scan:** All code steps contain complete code; `relay.example.com` and `<relay-id>`/`<public-iface>` are explicitly-declared deployment parameters (Global Constraints), not TODOs. No "handle edge cases"/"add validation" placeholders. ✅

**3. Type consistency:** `selectAdvertisedLocator`, `relayIDFromCircuit`, `findConfiguredRelay` signatures are identical in the Interfaces block, the implementation (Task 1 Step 3), and the tests (Task 1 Step 1). Call site in Step 4 matches the signature. Reuses existing `isCircuitAddr`/`externalAddr`/`constructedCircuitAddr` unchanged. ✅
