---
type: doc
project: depin
tags: [depin, identity, trust, mtls, signing]
created: 2026-07-05
---

# identity & trust — node identity, mTLS, and message signatures (start here)

How a DePIN node proves *who it is* (a cryptographic node ID + mTLS), and how the
system authorizes *what it may do* (signed order limits, orders, and piece hashes).
This is the onboarding guide: the trust model and what to expect. For the wire it runs
over see [`transport.md`](transport.md).

> Scope note: this document is about **node identity** (mTLS certificates) and
> **message signatures** (orders / piece hashes). It is **not** about data encryption.
> Where data-at-rest is mentioned, the system uses **server-side encryption** — the
> uplink is a server-side service that holds keys; there is no client-side or
> end-to-end key material in this model.

---

## 1. What it is & why

Two intertwined mechanisms:

- **Node identity (mTLS).** Every node owns a self-signed **CA** and a CA-signed
  **leaf** certificate plus its private key (`identity.FullIdentity`). Its **node ID**
  is derived deterministically from the CA's public key, so the ID and the certificate
  are cryptographically bound. mTLS handshakes verify the chain and can *pin* the
  expected peer node ID.
- **Message signatures.** Authorization to move data is carried by signed protobuf
  messages: a coordinator signs an **order limit**, the uplink signs an **order** and a
  **piece hash**, and at commit the coordinator verifies the piece-hash signature.
  Workers decide *which coordinators they trust* through a **trust pool**.

Why both? mTLS authenticates the *connection* (this socket is talking to node X). The
signatures authenticate *intent* across hops — an order limit a worker receives was
genuinely issued by a coordinator it trusts, even though the bytes arrived from an
uplink. Identity binds the actors; signatures bind the actions.

---

## 2. Mental model

```
                 ┌──────────── identity (per node) ─────────────┐
   self-signed CA ──signs──> leaf cert        node ID =
        │                       │            double-SHA256(CA pubkey)
        └── private Key ────────┘            + PoW difficulty (>= 36 trailing 0 bits)

   Connection:   client ──mTLS handshake──> server
                  VerifyPeerCertChains  (leaf signed by CA, CA self-signed)
                  + verifyIdentity(id)  (cert-derived node ID == expected)   [ckpt 1 & 5]

   Authorization (a PUT/GET):

     coordinator ──SignOrderLimit──────────────┐
                                                ▼
        uplink ──SignUplinkOrder (piece key)──> worker
        uplink ──SignPieceHash (piece key)────> worker
                                                │
                 worker verifies:               │
                   order-limit sig  via trust pool GetSignee + VerifyOrderLimitSignature [ckpt 2]
                   order sig        via piece public key  VerifyUplinkOrderSignature     [ckpt 3]
                                                │
                                                ▼
     coordinator (commit) ──VerifyPieceHashSignature(uplink)──>  accepted   [ckpt 4]
```

---

## 3. Glossary

| Term | Meaning | Where |
| ---- | ------- | ----- |
| **`FullIdentity`** | CA + leaf + private `Key` + node `ID` — a node's own identity | `pkg/identity/identity.go` |
| **`PeerIdentity`** | CA + leaf + `ID` — a *remote* node's identity (no key) | `pkg/identity/identity.go` |
| **node ID** | `double-SHA256(CA pubkey)`, version byte set; bound to the CA | `identity.go` `NodeIDFromKey` |
| **PoW difficulty** | trailing-zero-bit count of the node ID; CA gen retries until ≥ target | `certificate_authority.go` |
| **`Options` / `NewOptions`** | TLS options built from a `FullIdentity` + config | `internal/peertls/tlsopts` |
| **`ServerTLSConfig` / `ClientTLSConfig`** | `*tls.Config` for handshakes; client pins peer ID | `tlsopts/tls.go` |
| **`VerifyPeerCertChains`** | checks leaf is signed by CA and CA self-signed | `internal/peertls/peertls.go` |
| **`verifyIdentity(id)`** | checks the peer's cert-derived node ID equals expected | `tlsopts/tls.go` |
| **order limit** | coordinator-signed authorization a worker honors | `signing.SignOrderLimit` |
| **order** | uplink-signed (piece key) spend against an order limit | `signing.SignUplinkOrder` |
| **piece hash** | signed commitment to stored bytes | `signing.SignPieceHash` |
| **`Signer` / `Signee`** | sign / verify interfaces (`ID`, `HashAndSign`, `HashAndVerifySignature`) | `pkg/signing/sign.go`, `verify.go` |
| **`TrustService`** | worker's pool of approved coordinators | `worker/trust/pool.go` |
| **`signaturecheck.Check`** | worker's pluggable order/order-limit verifier (`Full`/`Trusted`/`AcceptAll`) | `worker/pkg/signaturecheck` |

---

## 4. Step-by-step walkthrough

### Identity: how a node gets its ID

`NewCA` (`pkg/identity/certificate_authority.go`) generates a self-signed CA whose
**node ID satisfies a PoW difficulty**: it loops generating keys (`GenerateKeys`) and
keeps one whose `id.Difficulty()` (count of trailing zero bits) meets `opts.Difficulty`
(default `36`). The CA then signs a leaf via `NewIdentity`, producing a
`FullIdentity{CA, Leaf, Key, ID}`. The ID itself comes from `NodeIDFromKey` →
`DoubleSHA256PublicKey` (it marshals the CA public key PKIX, hashes it with SHA-256
twice, takes 16 bytes, sets the version byte). Because the ID is a hash of the CA key,
you cannot present a cert whose ID you didn't grind for — that is the spam/Sybil cost.

Identities load from disk via `FullIdentityFromPEM` (chain + key) and remote peers via
`PeerIdentityFromChain` / `PeerIdentityFromContext`, both of which re-verify the leaf is
CA-signed.

### Checkpoint 1 & 5 — mTLS handshake (node identity)

`tlsopts.NewOptions(ident, cfg)` builds an `Options` carrying the node's cert.
`ServerTLSConfig()` and `ClientTLSConfig(id)` (`tlsopts/tls.go`) both:

- set `InsecureSkipVerify = true` (system root CAs are irrelevant — identity comes from
  node IDs, not a public PKI),
- install `VerifyPeerCertificate = peertls.VerifyPeerFunc(VerifyPeerCertChains, …)`
  which checks the leaf is signed by the CA and the CA self-signed
  (`verifyChainSignatures` / `verifyCertSignature` in `internal/peertls/utils.go`),
- and on the server set `ClientAuth = tls.RequireAnyClientCert` (so the handshake is
  **mutual** — the client must present a cert too).

When the dialer knows *which* node it expects (the transport's
`ClientTLSConfig(nodeURL.ID)`), it additionally installs `verifyIdentity(id)`: it pulls
the peer identity from the chain (`PeerIdentityFromChain`) and rejects the handshake if
`peer.ID != id`. This is the node-ID pin — checkpoint 1 (identity) enforced inside the
TLS handshake itself (checkpoint 5). Optionally `VerifyCAWhitelist` requires the peer's
CA to be signed by an admin CA when `UsePeerCAWhitelist` is configured.

### Checkpoint 2 — order-limit signature (coordinator → worker)

A coordinator authorizes work by `signing.SignOrderLimit(ctx, coordSigner, limit)`,
placing the signature in `OrderLimit.CoordinatorSignature` (encoded by
`EncodeOrderLimit`, which nulls the signature field before hashing). When a worker
receives the limit, `worker/piecestore/verification.go` enforces trust **fail-closed**:

```go
resolved, gErr := s.trust.GetSignee(ctx, limit.CoordinatorId)  // unknown coord → error
...
s.signatureCheck.VerifyOrderLimitSignature(ctx, resolved, limit)
```

`TrustService.GetSignee` (`worker/trust/pool.go`) only returns a `signing.Signee` if
`limit.CoordinatorId` is in the worker's approved set; it lazily resolves the
coordinator's `PeerIdentity` and wraps its leaf public key
(`signing.SigneeFromPeerIdentity`). The actual cryptographic check is
`signing.VerifyOrderLimitSignature` (reached through `signaturecheck.Full`). An unknown
coordinator never yields a signee, so the order is rejected before any signature math.

### Checkpoint 3 — order signature (uplink → worker, via piece key)

For each piece transfer the uplink signs an **order** with a per-transfer **piece
private key**: `signing.SignUplinkOrder(ctx, piecePrivKey, order)` →
`Order.UplinkSignature`. The worker verifies it with the matching **piece public key**
carried in the order limit, via `s.signatureCheck.VerifyUplinkOrderSignature(...)` →
`signing.VerifyUplinkOrderSignature` (`pkg/signing/verify.go`), which calls the piece
public key's `Verify`. This binds the order to the specific authorization, not to the
uplink's long-term identity.

### Checkpoint 4 — piece-hash signature (uplink → coordinator at commit)

On commit, the coordinator verifies the uplink's signed **piece hash**:
`coord/file/endpoint.go` calls `signing.VerifyPieceHashSignature(ctx, signee, hash)`.
`SignPieceHash` / `SignUplinkPieceHash` produced the signature (encoded by
`EncodePieceHash` over piece ID, hash, size, algorithm, timestamp). Workers likewise
sign piece hashes (`SignUplinkWorkerPieceHash`, verified by `VerifyWorkerPieceHash`) as
the audit/storage proof — see also `coord/audit/verifier.go`.

### Keeping the trust pool current

`worker/trust/NewPool(log, resolver, config, coordDB)` loads approved coordinators from
sources (HTTP / file / static), and `Refresh(ctx)` periodically reconciles the live set,
persisting addresses/status into `coordDB`. `VerifyCoordID`, `IsTrusted`, and
`GetNodeURL` are the read-side helpers; `GetSignee` is what verification actually uses.

---

## 5. Key types / entry points

```go
// identity (pkg/identity)
type FullIdentity struct { RestChain []*x509.Certificate; CA, Leaf *x509.Certificate; ID vo.UUID; Key crypto.PrivateKey }
type PeerIdentity struct { RestChain []*x509.Certificate; CA, Leaf *x509.Certificate; ID vo.UUID }
func NodeIDFromKey(pub crypto.PublicKey) (vo.UUID, error)        // double-SHA256(CA pubkey)
func PeerIdentityFromChain(chain []*x509.Certificate) (*PeerIdentity, error)

// mTLS (internal/peertls + tlsopts)
func NewOptions(i *identity.FullIdentity, c Config) (*Options, error)
func (o *Options) ServerTLSConfig() *tls.Config
func (o *Options) ClientTLSConfig(id vo.UUID) *tls.Config        // pins peer node ID
func VerifyPeerCertChains(_ [][]byte, chains [][]*x509.Certificate) error

// signing (pkg/signing)
func SignOrderLimit(ctx, coord Signer, *sharedpb.OrderLimit) (*sharedpb.OrderLimit, error)
func SignUplinkOrder(ctx, key vo.PiecePrivateKey, *sharedpb.Order) (*sharedpb.Order, error)
func SignPieceHash(ctx, signer Signer, *piecepb.PieceHash) (*piecepb.PieceHash, error)
func VerifyOrderLimitSignature(ctx, coord Signee, *sharedpb.OrderLimit) error
func VerifyUplinkOrderSignature(ctx, uplink vo.PiecePublicKey, *sharedpb.Order) error
func VerifyPieceHashSignature(ctx, signee Signee, *piecepb.PieceHash) error

// worker trust (worker/trust)
func NewPool(log, resolver IdentityResolver, cfg TrustConfig, coordDB CoordDB) (*TrustService, error)
func (p *TrustService) GetSignee(ctx, id vo.UUID) (signing.Signee, error)   // fail-closed
func (p *TrustService) VerifyCoordID(ctx, id vo.UUID) error
func (p *TrustService) Refresh(ctx) error
```

Enforcement sites: `worker/piecestore/verification.go` (ckpt 2 & 3),
`coord/file/endpoint.go` (ckpt 4), `internal/peertls/tlsopts/tls.go` (ckpt 1 & 5).

---

## 6. Gotchas / edge cases

- **`InsecureSkipVerify` is correct here.** It disables *system root CA* validation
  precisely because identity is the node ID, not a public PKI. The real check is in
  `VerifyPeerCertificate`. Do not "harden" it by re-enabling root verification.
- **Trust is fail-closed by construction.** `GetSignee` errors for an unknown
  coordinator, so verification aborts *before* the signature is examined — the absence
  of a signee is the rejection.
- **`signaturecheck` has a no-op mode.** `signaturecheck.AcceptAll` (used in tests)
  skips signature verification entirely. Make sure production wiring uses `Full` /
  `Trusted`, not `AcceptAll`.
- **Order signatures use a per-transfer piece key, not the uplink's node key.** The
  verifying key (`vo.PiecePublicKey`) rides in the order limit; this is separate from
  mTLS identity.
- **PoW difficulty is set once at CA generation.** A node ID with insufficient
  difficulty cannot be retrofitted — you regenerate the CA (and thus the ID).
- **Encoders null the signature field before hashing.** `EncodeOrderLimit` /
  `EncodePieceHash` zero the signature so signing and verifying hash identical bytes;
  re-ordering or skipping that step breaks verification.
- **Server-side encryption only.** Any key material for data-at-rest lives in the
  server-side uplink; this trust model deliberately carries no client-held data keys.

---

## 7. Where to go next

- [`../pkg/identity`](../pkg/identity) — `FullIdentity`, CA generation, node-ID derivation.
- [`../internal/peertls`](../internal/peertls) — mTLS handshake and chain verification.
- [`../pkg/signing`](../pkg/signing) — order / order-limit / piece-hash signing + verify.
- [`../worker/trust`](../worker/trust) — the worker's approved-coordinator pool.
- [`transport.md`](transport.md) — where `ClientTLSConfig(id)` is used to dial.
- [`configuration.md`](configuration.md) — how identity and TLS options are wired in.
- [`../SOURCE_STRUCTURE.md`](../SOURCE_STRUCTURE.md) — the layered peer architecture.
