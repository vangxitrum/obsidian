---
type: doc
project: depin
tags: [depin, worker, onboarding, identity, check-in]
created: 2026-07-06
---

# Worker Onboarding

How a new storage worker (`aioznode`) goes from bare machine to an eligible node
receiving piece uploads. Covers identity, authorization, startup, check-in, and
overlay eligibility.

The design mirrors Storj: node-ID-based mutual TLS, no central PKI root of
trust, coordinator-issued eligibility.

---

## 0. Mental model

A worker's identity is a two-cert chain:

```
CA cert (long-lived, defines node ID)  ->  leaf cert (working key, used every handshake)
```

- **Node ID** = double-SHA256 of the CA public key, with a version byte. Must
  satisfy a target **difficulty** (number of trailing zero bits), mined at
  create time.
- The worker trusts the coordinator, and the coordinator trusts the worker,
  purely by validating this chain over mTLS. There are no CA-signed roots
  (`InsecureSkipVerify: true` in the TLS config, on purpose).
- In **release** builds the coordinator additionally enforces a **peer-CA
  whitelist**: a worker's CA must be signed by an authority CA on the
  coordinator's whitelist. This is what the authorization step (section 2)
  produces. In **dev** builds the whitelist is off, so unauthorized identities
  connect freely.

Four phases:

1. **Create identity** — mine CA, issue leaf.
2. **Authorize** — get the CA counter-signed by the operator's signing authority
   (release only).
3. **Configure + start** — load identity, fetch relays, boot libp2p host.
4. **Check in** — register with the coordinator; become eligible for uploads.

---

## 1. Create identity

Command (`cmd/keytool`):

```bash
keytool create worker1 --identity-dir dev --difficulty 36 --concurrency 4
```

`cmd/keytool/cmd_identity.go:35` (`runCreate`, `:69`):

1. **Mine CA** — `identity.CASetupConfig.Create` grinds a CA key until its node
   ID has `difficulty` trailing zero bits. Writes `ca.cert`, `ca.key`.
2. **Issue leaf** — `ca.NewIdentity()` (`pkg/identity/certificate_authority.go:409`)
   generates a fresh leaf key and signs a leaf cert with the CA key. Writes
   `identity.cert`, `identity.key`.

Output dir `<identity-dir>/<service>/` holds four files:

```
ca.cert  ca.key  identity.cert  identity.key
```

`identity.cert` is the concatenated chain (`chain[0]=leaf`, `chain[1]=CA`);
there is no separate CA load at worker boot — the CA travels inside the identity
chain.

### Cert templates (no OpenSSL)

Templates are Go structs, not `.cnf`/`.ext` files (the repo has none).
`internal/peertls/templates.go`:

- **CA** (`:9`): `KeyUsage: CertSign`, `IsCA: true`.
- **Leaf** (`:27`): `KeyUsage: DigitalSignature | KeyEncipherment`,
  `ExtKeyUsage: ServerAuth + ClientAuth` (same cert serves as both TLS client and
  server for mTLS), `IsCA: false`.

`ca.NewIdentity(exts ...pkix.Extension)` accepts extra X.509 extensions (e.g. a
revocation extension) baked into the leaf before signing — the programmatic
equivalent of an OpenSSL `.ext` file. `keytool create` passes none.

---

## 2. Authorize the identity (release builds)

Turns a freshly-created, untrusted CA into one the coordinator's whitelist
accepts. Two variants: remote (over the network) and local (offline).

### 2a. Operator side — run a signing authority

Signing lives behind the `signserver` build tag
(`cmd/keytool/cmd_signserver.go:1`); default builds get a no-op stub.

```bash
# mint single-use tokens for an operator email
keytool auth create ops@aioz.io --count 5        # prints token strings

# run the signing gRPC server (mTLS)
keytool sign-server --address 127.0.0.1:8888 \
    --auth-db <identity-dir>/authorizations.db --min-difficulty 36
```

- `auth create` (`:148`) mints tokens via `authorizationDB.Create`. A **token**
  is `{UserID(email), Data[64]random}`, serialized `userID:base58(data)`
  (`certificate/authorization/authorizations.go:52,143`).
- `auth export <email>` (`:185`) lists open vs claimed tokens (with the node IDs
  that claimed them).
- The server loads the **authority FullCA** (`ca.cert`+`ca.key`) and registers
  `certificate.NewEndpoint(log, signerCA, authDB, minDifficulty)` (`:114`).
- Token DB is bbolt-backed, bucket `"authorizations"`, keyed by email
  (`certificate/authorization/db.go`).

### 2b. Worker side — redeem a token

```bash
keytool authorize worker1 <auth-token> --signer.address=127.0.0.1:8888
```

`cmd/keytool/cmd_authorize.go` (`runAuthorize`, `:54`):

1. Load the node's CA + leaf from `<identity-dir>/worker1/`.
2. Build mTLS options with **whitelist off** (`signerTLSConfig`, `:16`) — the
   worker's CA isn't trusted yet, so it connects with an unverified leaf.
3. `certclient.New(...).Sign(ctx, authToken)` sends
   `SigningRequest{AuthToken, Timestamp}` (`certificate/certclient/client.go:45`).
4. Splice the returned DER chain back in via `applySignedChain`
   (`cmd_sign.go:100`): backs up originals, sets `ca.Cert = signedChain[0]`,
   `ca.RestChain = signedChain[1:]`, re-saves `ca.cert` and `identity.cert`.

### 2c. Server endpoint — sign + claim

`certificate/endpoint.go` `Sign` (`:46`):

```go
peerIdent, _ := identity.PeerIdentityFromContext(ctx)   // from the mTLS leaf
signedPeerCA, _ := endpoint.ca.Sign(peerIdent.CA)       // authority signs worker CA
signedChainBytes := [][]byte{signedPeerCA.Raw, endpoint.ca.Cert.Raw}
signedChainBytes = append(signedChainBytes, endpoint.ca.RawRestChain()...)
difficulty, _ := peerIdent.ID.Difficulty()
endpoint.authorizationDB.Claim(ctx, &authorization.ClaimOpts{
    TokenStr: req.AuthToken, NodeID: peerIdent.ID.String(),
    Difficulty: uint16(difficulty), Addr: peerAddr(ctx),
    Timestamp: req.Timestamp, ChainBytes: signedChainBytes,
    MinDifficulty: endpoint.minDifficulty,
})
return &certpb.SigningResponse{Chain: signedChainBytes}, nil
```

`Claim` (`certificate/authorization/db.go:86`) enforces single-use and validity:

- Timestamp within `MaxClockSkew = 5m`.
- `Difficulty >= MinDifficulty`.
- Token exists and is not already claimed (`ErrNotFound` / `ErrAlreadyClaimed`).

### 2d. Offline variant

```bash
keytool sign-ca worker1        # signer key on the same machine, no network
```

`cmd/keytool/cmd_sign.go:13` — `signer.Sign(ca.Cert)` then the same
`applySignedChain`.

### 2e. How the CA becomes coordinator-trusted

- Verification: `internal/peertls/peertls.go:74` `VerifyCAWhitelist` — the peer's
  CA must be signed by a whitelist CA, else `ErrVerifyCAWhitelist`.
- Wiring: `internal/peertls/tlsopts/options.go:89` `configure()` registers the
  whitelist verify func when `UsePeerCAWhitelist`.
- Config default (`internal/peertls/tlsopts/config.go:9`): `UsePeerCAWhitelist`
  `releaseDefault:"true"`, `devDefault:"false"`; `PeerCAWhitelistPath` overrides
  the compiled-in `DefaultPeerCAWhitelist` (`tlsopts/cert.go`).
- The coordinator builds TLS via `tlsopts.NewOptions(...)` without forcing the
  flag off (`coord/peer.go:132,180`), so release builds enforce the whitelist.
  Parties that turn it **off**: the worker's own outbound TLS
  (`worker/peer.go:166`), the authorize client (`cmd_authorize.go:22`), and the
  uplink SDK (`uplink-sdk/client.go:95`).

**End to end:** operator `auth create` -> token handed to worker operator ->
worker `keytool authorize` -> authority signs the worker CA and atomically
claims the token with the node ID -> worker splices the signed chain -> the
worker's CA now chains to a whitelist authority and the coordinator accepts it.

---

## 3. Configure and start the worker

Binary `cmd/worker` (root command `aioznode`), subcommands `run`, `api`,
`setup` (`main.go:37`). `api` is the real "start" command (`command.go:13`).

Default dirs (`main.go:26`):

- config: `fpath.ApplicationDir("aioznode")`
- identity: `fpath.ApplicationDir("aioznode", "identity", "worker")`

Override with `--config-dir` / `--identity-dir`. Config is `worker.Config`
(`worker/peer.go:49`): `Contact`, `Storage`, `Identity`, `Server`,
`RelayCachePath` (`${CONFDIR}/relay-cache.json`), `RelayNumRelays`.

### Config generation

- `worker setup` (`cmd/worker/start.go:13`): creates identity + config dirs and
  writes `config.yaml` via `process.SaveConfig`.
- `worker config storage --limit ... --config-path ...`
  (`cmd/worker/config_tool.go:16`): edits `node.storage-limit` in an existing
  yaml (min 15 GiB).
- Dev fleet: `dev/spawn-workers.sh` generates `dev/worker1`-style dirs
  (`gen_worker_identity` `:191`, `gen_worker_config` `:327`), copies
  `trust-cache.json` so the coordinator URL is known before the trust source
  responds, and launches `bin/worker api --config-dir <worker-dir>`.

> Per project convention, prefer `dev/worker1/` + the setup command over
> hand-writing config files.

### Boot sequence (`worker api` -> `api.go:19`)

```
apiConfig.Identity.Load()            # read identity.cert + identity.key
  -> FullIdentityFromPEM: split leaf/CA/RestChain, derive node ID
open + migrate worker DB
worker.NewPeer
  if behind NAT (Server.HavePublicAddress == false):
      trustPool.Refresh(ctx)
      relay.ResolveRelayAddrs(...)   # GetRelayAddrs RPC, section 4
      build libp2p host w/ StaticRelays (AutoRelay fixed at construction)
contact.NewService / NewChore        # per-coord check-in cycle, section 5
```

Identity load (`pkg/identity/identity.go:142`) reads the cert chain + key and
builds a `FullIdentity` (`Leaf`, `CA`, `RestChain`, `ID`).

---

## 4. Relay bootstrap (behind-NAT workers)

A worker with no routable address needs relay multiaddrs *before* the libp2p
host is built, because AutoRelay must be fixed at host construction.

`worker/relay/bootstrap.go`:

- `FetchRelayAddrs` (`:27`) calls `GetRelayAddrs` on a coordinator:

  ```go
  resp, _ := contactpb.NewContactServiceClient(conn).
      GetRelayAddrs(ctx, &contactpb.GetRelayAddrsRequest{})
  return resp.RelayAddrs, nil
  ```

- `ResolveRelayAddrs` (`:55`) tries each trusted coord (first success wins),
  refreshes the on-disk cache (`${CONFDIR}/relay-cache.json`), and falls back to
  cache if all coords are unreachable.
- Invoked in `NewPeer` before the host, only when behind NAT
  (`worker/peer.go:198`).
- Coordinator serves the operator-configured fleet
  (`coord/contact/endpoint.go:49`), from `Config.RelayAddrs`
  (`coord/contact/service.go:46`). The same list is echoed in every
  `CheckInResponse.relay_addrs` and refreshes the cache via `OnCheckInRelayAddrs`
  (`bootstrap.go:110`).

---

## 5. Check-in / registration

Proto `pkg/pb/coord/contact/v1/contact.proto`:

```proto
service ContactService {
    rpc CheckIn (CheckInRequest) returns (CheckInResponse);
    rpc GetRelayAddrs (GetRelayAddrsRequest) returns (GetRelayAddrsResponse);
}
message CheckInRequest {
    bytes id = 1;              // node ID
    string address = 2;        // external / relay addr
    string version = 3;
    SystemInfo system_info = 4;
    DiskSpace disk_space = 5;  // capacity
    int64 debounce_limit = 6;
}
message CheckInResponse {
    bool ping_peer_success = 2;
    string ping_peer_error_msg = 3;
    repeated string relay_addrs = 4;
}
```

### Worker side

`worker/contact/service.go:155` (`pingCoordOnce`) calls `CheckIn` with
`self := service.Local()` (ID, Address, Version, SystemInfo, DiskSpace,
DebounceLimit). Node ID + external address are set at wiring time
(`peer.go:327`); capacity is refreshed by the monitor service via `UpdateSelf`
(`service.go:238`, from `worker/monitor/service.go:337`).

**Scheduling** — the contact chore runs one ping cycle per trusted coordinator
(`worker/contact/chore.go:91`), with a 1-minute cycle to add/remove coords:

- `Interval`: `releaseDefault:"1h"`, `devDefault:"30s"` (`service.go:41`).
- `CheckInTimeout`: `releaseDefault:"10m"`, `devDefault:"15s"`.
- Retry/backoff within a cycle doubles from `1s` up to `maxInterval`
  (`PingCoordinator`, `service.go:115`).

### Coordinator side

`coord/contact/endpoint.go:57` (`CheckIn`):

1. `identity.PeerIdentityFromContext(ctx)` — node ID derived from the **CA cert**
   over mTLS (`pkg/identity/identity.go:178,242`). The worker cannot assert its
   own identity fields.
2. Rate-limit per node ID (`:67`).
3. Capture the leaf public key for later piece-hash verification (`:74`).
4. Build a `Worker` from `req` fields (`:79`); if the reported address is empty,
   fall back to the observed gRPC source address (`:92`).
5. Delegate to `ContactUsecase.CheckIn` (`coord/contact/service.go:155`):
   read prior state, **ping-back dial** the worker's claimed address to verify
   reachability (`PingBack`, `:97`), advance uptime ring buffers, server-derive
   IP/subnet/country via GeoIP, set `IsOnline`/`LastSeen`, then
   `store.AddWorker`.
6. Echo ping-back result + relay addrs in the response (`:110`).

A worker whose ping-back fails is recorded `IsOnline=false` and is **not**
offered for uploads.

Identity-level checks (difficulty, whitelist, revocation) happen at the TLS
layer, not in `CheckIn` — see `tlsopts/options.go:89` and section 2e.

### Worker record

`coord/db/worker_repo.go` (`WorkerRepository` implements `contact.Store`).
Entity `contact.Worker` (`coord/contact/worker.go:10`, table `workers`). Fields
(schema owned by SQL migrations, `AutoMigrate` off):

- Identity: `id` (UUID PK), `address`, `public_key`, `alias` (BIGSERIAL, compact
  int used in segment piece lists; gorm read-only so re-check-ins never
  overwrite it).
- `system_info_*`, `disk_space_*` (capacity).
- Liveness: `uptime_day|week|month` ring buffers, `last_contacted_at`,
  `last_seen`, `is_online`.
- Network/geo (server-derived, never node-asserted): `resolved_ip`, `last_net`,
  `country_code`, `egress_nets`.
- Reputation/audit: `audit_success`, `audit_failure`, `audit_history`,
  `vetted_at`.
- Disqualification/suspension: `disqualified_at`, `disqualification_reason`,
  `suspended_at`, `unknown_audit_suspended_at`, `offline_suspended_at`.

`AddWorker` upserts via gorm `Save` keyed by UUID.

---

## 6. Becoming eligible for uploads (overlay)

Registration alone doesn't earn traffic. Selection reads a slim projection
`contact.SelectedWorker` (`coord/contact/selectedworker.go:20`, mirrors Storj
`nodeselection.SelectedNode`): `Vetted`, `Suspended`, `Online`,
`AuditSuccessRate`, `OnlineScore`, `FreeDisk`, `LastNet`, `CountryCode`.

Read paths (`coord/db/worker_repo.go`):

- `selectWorkerRows` (`:220`): base pool = `disqualified_at IS NULL` AND
  `deleted_at IS NULL` AND `is_online = true`, plus a
  `last_seen >= now - onlineWindow` liveness gate.
- `SelectParticipatingWorkersSplit` (`:325`): upload pool, **excludes
  suspended**, partitions `reputable` (`Vetted`) vs `new` (`!Vetted`).
- `SelectDownloadableWorkers` (`:314`): download pool **includes suspended**.
- `projectWorker` (`:351`): folds audit counters + history into scores;
  `Vetted = VettedAt != nil`; `Suspended = suspended_at OR
  unknown_audit_suspended_at OR offline_suspended_at`.

Selection cache: `coord/overlay/uploadselection.go` `UploadSelectionCache` —
periodically refreshed per-placement snapshot split reputable/new, applies
`WorkerFilter`/`UploadFilter`, then per-request new-node fraction, network
diversity, exclusions.

Filters (`coord/placement/worker_filters.go`): `ReputationFilter.Match` requires
`OnlineScore >= MinOnlineScore` (`:35`); plus `CountryFilter`, `CompositeFilter`,
`OrFilter`.

Disqualification (gates selection, `coord/db/overlay_repo.go`):
`DisqualifyWorker` (`:28`), `UndisqualifyWorker` (`:41`),
`DQWorkersLastSeenBefore` (stray/offline-too-long DQ, `:57`). Reputation
subsystem driving vetting/suspension: `coord/reputation/`.

A **new** node is eligible immediately but only for the new-node fraction of each
upload; it becomes `Vetted` after enough successful audits, moving into the
reputable pool.

---

## 7. Quick reference

| Step | Command | Writes / effect |
|------|---------|-----------------|
| Create identity | `keytool create <svc> --difficulty 36` | `ca.cert ca.key identity.cert identity.key` |
| Operator: mint token | `keytool auth create <email> --count N` | token strings |
| Operator: run signer | `keytool sign-server --address ... --auth-db ...` | signing gRPC (needs `signserver` build tag) |
| Authorize (remote) | `keytool authorize <svc> <token> --signer.address=...` | signed CA chain spliced in |
| Authorize (offline) | `keytool sign-ca <svc>` | same, local signer |
| Configure | `worker setup` / `dev/spawn-workers.sh` | `config.yaml`, dirs, trust cache |
| Start | `worker api --config-dir <dir>` | load identity, relays, check-in loop |

### Dev vs release

| | dev | release |
|--|-----|---------|
| Peer-CA whitelist | off | **on** (authorization required) |
| Check-in interval | 30s | 1h |
| Check-in timeout | 15s | 10m |

---

*Sources: `cmd/keytool/*`, `cmd/worker/*`, `worker/*`, `coord/contact/*`,
`coord/db/worker_repo.go`, `coord/overlay/*`, `coord/placement/*`,
`internal/peertls/*`, `certificate/*`, `pkg/identity/*`.*
