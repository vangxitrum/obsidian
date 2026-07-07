---
type: doc
project: depin
tags: [depin, dataflow, audit, proof-of-storage]
created: 2026-07-05
---

# dataflow: audit — end-to-end (the proof-of-storage path)

How the coordinator forces every worker to *prove* it still holds the encrypted pieces it promised to
store. This is a **cross-cutting** guide: it follows one audit through every process it touches — the
coordinator's **Core** role (selection) and **Audit** role (verification), and the storage `worker`s
that answer the challenges — naming the real packages and functions at each hop. For the
subsystem-local guide see [`../coord/audit`](../coord/audit) (the package README is the canonical
source for this pipeline).

> **What audit operates on is server-side-encrypted data.** Workers store only encrypted pieces and
> never hold keys; the audit never decrypts anything — it checks *possession of bytes*, not plaintext.
> The bytes were encrypted by the **uplink service** at upload time (server-side AES-GCM; see
> [`dataflow-upload.md`](dataflow-upload.md)), so this is **not** an end-to-end / zero-knowledge
> scheme — the operator can read data, but the audit only proves storage.

---

## 1. Summary

Each round the coordinator's **Core** role samples committed segments per worker using a **weighted
reservoir** ([`audit.Observer.BuildQueue`](../coord/audit/observer.go)), and the
[`Producer`](../coord/audit/producer.go) pushes them into a durable DB `VerifyQueue`. Scale-out
**Audit** processes each run a [`Worker`](../coord/audit/worker.go) that claims rows (`FOR UPDATE SKIP
LOCKED`), loads the segment, and calls [`Verifier.Verify`](../coord/audit/verifier.go). Verification
dispatches on the redundancy algorithm:

- **Reed-Solomon** (`verifyReedSolomon`): pick a random **stripe**, mint coordinator-signed GET order
  limits for one share per piece, fetch shares in parallel, run FEC `Correct` over **copies** via
  [`pkg/infectious`](../pkg/infectious), and flag any share that disagrees with the canonical
  reconstruction.
- **CLONE** (`verifyClone`): pick a random **chunk**, fetch each replica's chunk + Merkle proof, and
  check it against the stored root with [`merkle.VerifyProof`](../pkg/merkle).

The [`Reporter`](../coord/audit/reporter.go) folds outcomes into per-worker counters and online-score
windows ([`audithistory.go`](../coord/audit/audithistory.go)). Ambiguous results are **contained** and
re-checked by the [`ReverifyWorker`](../coord/audit/reverifyworker.go) before any blame is assigned.

---

## 2. The whole path

```
        CORE role (singleton)                    │            AUDIT role (scale-out, N processes)
        ─────────────────────                    │            ────────────────────────────────────
                                                 │
 ┌─────────┐   BuildQueue   ┌──────────┐         │   Next()      ┌────────┐   Verify()   ┌──────────┐
 │ Store   │ ─────────────▶ │ Observer │         │  (SKIP        │ Worker │ ───────────▶ │ Verifier │
 │ segment │   weighted     │ per-     │         │   LOCKED)     │ claim+ │              │ dispatch │
 │ scan    │   reservoir    │ worker   │         │               │ load+  │              │ on Algo  │
 └─────────┘                └────┬─────┘         │               │ record │              └────┬─────┘
                                 │               │               └───┬────┘                   │
                            ┌────▼─────┐  Push   ╔═══════════════╗   │ contained              │
                            │ Producer │ ──────▶ ║  VerifyQueue  ║ ──┘ pieces                 │
                            │ per-round│ batches ║  (DB)         ║                            │
                            └──────────┘         ╚═══════════════╝                            │
                                                 │                                            │
                                                 │     ┌──────────────────────────────────────┤
                                                 │     │ Path A: REED_SOLOMON                  │ Path B: CLONE
                                                 │     ▼                                       ▼
                                                 │  pick random stripe                    pick random chunk
                                                 │  CreateGetOrderLimits ──┐              CreateGetOrderLimits ──┐
                                                 │   └ SignOrderLimit       │              └ SignOrderLimit       │
                                                 │  fetchShares (parallel)  │              fetchProofs (parallel) │
                                                 │   └ signSingleUseOrder   │               └ signSingleUseOrder │
                                                 │            │             ▼                         │          ▼
                                                 │            │        WORKER Download          WORKER GetChunkProof
                                                 │            │     • verifyOrderLimit(GET)      • verifyOrderLimit(GET)
                                                 │            │       (coord sig via trust)      • verifyOrder (uplink)
                                                 │            │     • verifyOrder (uplink sig)   • merkle.Build over
                                                 │            │     • usedserials replay           stored bytes → Proof
                                                 │            ▼
                                                 │   infectious FEC Correct (copies)        merkle.VerifyProof(
                                                 │   → compare shares → corrupted?            chunk, idx, proof, root)
                                                 │            │                                       │
                                                 │            └──────────────┬────────────────────────┘
                                                 │                           ▼
                                                 │                     classify → AuditReport
                                                 │                           │
   ┌──────────┐  AddAuditCounts   ┌──────────┐   │                           ▼
   │ ReportDB │ ◀──────────────── │ Reporter │ ◀─┼──────────────────────  Reporter.Record
   │ counters │  + online window  │          │   │                           │ PendingAudits (contained)
   │ +online  │                   └──────────┘   │                           ▼
   └──────────┘                                  │                   ╔═══════════════╗   GetNextJob()
                                                 │                   ║ ReverifyQueue ║ ──────────────┐
                                                 │                   ║ (DB)          ║               ▼
                                                 │                   ╚═══════════════╝       ┌────────────────┐
                                                 │                                           │ ReverifyWorker │
                                                 │                          recheck ONE piece│ ReverifyPiece  │
                                                 │   Success/Failure/ClientFault ◀───────────│ (RS: whole pc; │
                                                 │                                           │  CLONE: chunk) │
                                                 │                                           └────────────────┘
```

The two `╔═╗` boxes are durable DB queues — the only hand-off between the singleton Core and the
scale-out Audit side, and the reason claims survive restarts and never double-process.

---

## 3. Glossary

| Term | Meaning | Where |
| ---- | ------- | ----- |
| **segment** | one committed remote object slice with a piece list; the unit of audit | `store.go` `SegmentRow` / `placement.AuditSegment` |
| **piece** | one worker's RS share or CLONE replica of a segment | `placement.AuditPiece` |
| **reservoir** | weighted per-worker random sample (larger segments weighted higher) | `observer.go` / `placement.ReservoirSet` |
| **`VerifyQueue`** | DB queue of segments awaiting audit; `Next()` claims `FOR UPDATE SKIP LOCKED` | `store.go` `VerifyQueue` |
| **`ReverifyQueue`** | DB containment queue of (segment, worker, piece) re-check jobs | `store.go` `ReverifyQueue` |
| **stripe** | one RS row across all pieces; the audited unit for Reed-Solomon | `verifier.go` `verifyReedSolomon` |
| **chunk** | one Merkle leaf's worth of bytes; the audited unit for CLONE | `verifier.go` `verifyClone` |
| **outcome** | per-worker verdict: Success / Failure / Offline / Unknown / Contained / ClientFault | `placement.AuditOutcome` |
| **contained** | worker dialed but stalled/ambiguous → re-checked before judgement | `verifier.go` `classifyErr` |
| **client fault** | worker bytes intact but stored Merkle root wrong → file flagged CORRUPTED | `verifier.go` `reverifyCloneChunk` |
| **online score** | per-window average of online/total audits (online = any non-offline outcome) | `audithistory.go` `OnlineScore` |
| **single-use order** | uplink-signed order paired with the coordinator-signed GET limit | `fetch.go` `signSingleUseOrder` |

---

## 4. Step-by-step walkthrough

### 4.1 Selection — Core, once per `Config.Interval`

[`Producer.Run`](../coord/audit/producer.go) calls
[`Observer.BuildQueue`](../coord/audit/observer.go). `BuildQueue` forks a parallel scan over
`Store.IterateAuditSegments` (every committed remote segment). Each fork fills a private
`placement.ReservoirSet`: for every segment it offers the target to **every worker that holds a
piece** via `set.Sample(workerAlias, rng, target)` — a **weighted reservoir** (`ReservoirSlots`
default 3), so each worker is tested regularly and larger segments proportionally more often. Forks
`Merge` into a master set, which `Flatten`s to a deduplicated `[]AuditSegment`. `Producer` maps these
to `QueueSegment`s (id + size + expiry) and `Push`es them to the `VerifyQueue` in
`QueuePushBatchSize` batches.

### 4.2 Claim & dispatch — Audit role

[`Worker.Run`](../coord/audit/worker.go) loops `queue.Next(ctx)` — a row-level `FOR UPDATE SKIP
LOCKED` claim, so N processes partition the queue with no double-claims (`ErrQueueEmpty` → poll-sleep).
Each claimed segment goes to `auditOne` on a bounded concurrency limiter: it skips already-expired rows
(expiry is denormalized onto the row), loads the full target via `SegmentLoader.GetAuditSegment` (a
`vo.ErrRecordNotFound` means deleted/repaired → drop), then calls
[`Verifier.Verify`](../coord/audit/verifier.go), which dispatches on `seg.Redundancy.Algorithm`.

### 4.3 Path A — Reed-Solomon (`verifyReedSolomon`)

1. **Stripe geometry:** `stripes = ceil((encryptedSize + 4) / (shareSize*required))`; pick a random
   `stripeIndex`; the audited share is bytes `[stripeIndex*shareSize, +shareSize)` of every piece. The
   `+4` is the `stripeLengthPrefix` the encoder pads on (see [`../coord/file`](../coord/file)
   `RedundancyScheme.PieceSize`).
2. `resolveWorkers` maps aliases → `contact.Worker`; unresolved (offline/removed) aliases are skipped.
3. [`OrderMinter.CreateGetOrderLimits`](../coord/orders) (= `order.Service.CreateGetOrderLimits`) mints
   **coordinator-signed GET order limits** authorizing exactly one share (`shareSize` bytes) per piece,
   returning an ephemeral `vo.PiecePrivateKey`.
4. `fetchShares` downloads each share in parallel via `Fetcher.Fetch` ([`fetch.go`](../coord/audit/fetch.go)).
   Each fetch signs the **single-use uplink order** (`signSingleUseOrder`, with the ephemeral piece
   key) alongside the coordinator limit, dials the worker, and reads the bounded range from the
   worker's [`Download`](../worker/piecestore) endpoint. Transport errors classify per `classifyErr`
   (NotFound → Failure, timeout → Contained, Unavailable/dial → Offline). A short/truncated share is
   corruption → Failure.
5. If `len(good) >= required`, `auditShares` runs FEC `Correct` over **copies** of the returned shares
   (via [`../pkg/infectious`](../pkg/infectious) / `ers_schema.NewFEC`), reconstructs the canonical
   value of every share, and flags any original share that disagrees as `corrupted`. RS being MDS, any
   `required` honest shares uniquely determine all shares. If FEC can't decode or too few shares
   return, successful responders are *downgraded to Unknown* — never a false pass.
6. `classify` folds responses into the `AuditReport`: clean responders → `Successes`, mismatched →
   `Failures`, plus `Offlines`, `PendingAudits` (contained), `Unknown`.

### 4.4 Path B — CLONE (`verifyClone`)

1. Require a stored `MerkleRoot` and `ChunkSize` (else `ErrNoMerkleRoot`). Compute
   `numLeaves = merkle.NumLeavesFor(encryptedSize, chunkSize)` and pick a random `chunkIndex`.
2. Resolve workers and mint GET limits for one chunk (`ChunkSize` bytes) per **replica** — CLONE
   pieces are full copies, not erasure shares.
3. `fetchProofs` calls `Fetcher.FetchChunkProof` per replica → `{Chunk, Proof, NumLeaves}`. On the
   worker, [`PieceStoreEndpoint.GetChunkProof`](../worker/piecestore/endpoint.go) re-reads its stored
   bytes, rebuilds the tree on demand with `merkle.Build`, and returns `tree.Proof(idx)` plus the
   chunk.
4. For each response, [`merkle.VerifyProof(chunk, chunkIndex, proof, MerkleRoot)`](../pkg/merkle)
   checks the inclusion proof against the **stored root** the coordinator holds. No reconstruction, no
   quorum — one responding copy gives a definitive per-worker verdict. A mismatch is *ambiguous*
   (corrupt copy vs. bad client-committed root), so it becomes **Contained**, deferring blame to
   reverification.

### 4.5 Record — Audit role

`auditOne` calls [`Reporter.Record`](../coord/audit/reporter.go), which `aggregate`s the report into
one `WorkerAuditDelta` per worker; `ReportStore.AddAuditCounts` adds them to running counters
(additive, never replaced) plus the **online windows** ([`audithistory.go`](../coord/audit/audithistory.go)
`AddToHistory` / `OnlineScore`). Then every `report.PendingAudits` piece is `ReverifyQueue.Insert`ed
(idempotent per segment+worker, preserving accrued `reverify_count`).

### 4.6 Reverification — Audit role, the containment path

[`ReverifyWorker.Run`](../coord/audit/reverifyworker.go) claims jobs via
`ReverifyQueue.GetNextJob(retryInterval)` (oldest job past its retry interval; stamps `last_attempt`,
increments `reverify_count`). `processJob` loads the segment and calls `Verifier.ReverifyPiece`, which
re-checks **exactly one** contained piece:

- **RS** (`reverifyRSPiece` → `fetchAndVerifyWholePiece` → `verifySignedPiece`): download the *whole*
  stored piece plus the worker's signed `PieceHeader` (worker's
  [`GetSignedPiece`](../worker/piecestore/endpoint.go) endpoint, `FetchSignedPiece`), then verify the
  full chain — (1) the stored limit is for the very piece we authorized, (2) the **coordinator** signed
  that original PUT order limit, (3) the **uplink** signed the piece hash, (4) the downloaded bytes
  recompute (SHA-256) to that hash. The coordinator stores no per-piece reference: the worker's own
  signed header is the source of truth.
- **CLONE** (`reverifyCloneChunk`): fetch a fresh random chunk proof; if it verifies → Success.
  Otherwise escalate by fetching the whole copy + its uplink-signed hash (`fetchAndVerifyWholePiece`):
  data intact but proof failed ⇒ **`AuditClientFault`** (the *client* committed a bad root); data
  corrupt ⇒ **`AuditFailure`**.

`processJob` then resolves the outcome: `Success` → `Remove`; `Failure` → `recordFailure` + `Remove`;
`ClientFault` → `FileFlagger.MarkFileCorruptedBySegmentID` (flag the file CORRUPTED so downloads
reject it — one bad segment makes the file unrecoverable) + `Remove`; `Contained` → fails once
`ReverifyCount` reaches `MaxReverifyCount`, else left for retry; offline/unknown → retry, bounded by
`maxReverifyAttempts` (10). Being DB-backed, jobs survive restarts and are shared across processes.

---

## 5. Verification & signature checkpoints

Every download a verifier issues carries **three** layers the worker independently checks, plus a
fourth during reverification. These are exactly the same checks the upload path established, now read
back ([`worker/piecestore/verification.go`](../worker/piecestore/verification.go)):

| # | Checkpoint | Signed by | Verified by (worker) | Where |
| - | ---------- | --------- | -------------------- | ----- |
| 1 | **GET order limit** (action/addressee/expiry + signature) | coordinator (`SignOrderLimit`) | `verifyOrderLimit(GET)` via the coordinator **trust pool** (`trust.GetSignee` + `VerifyOrderLimitSignature`), fail-closed | `verification.go` |
| 2 | **Single-use serial replay** | — | `usedSerials.Add` (serial recorded; a replayed limit is rejected) | `verification.go` |
| 3 | **Single-use order** | uplink/audit (`signSingleUseOrder` with the ephemeral piece key) | `verifyOrder` (`VerifyUplinkOrderSignature`, serial matches the limit) | `verification.go` / `fetch.go` |
| 4 | **Signed-piece chain** *(reverify only)* | coordinator (PUT limit) + uplink (piece hash) | coordinator `verifySignedPiece`: coord sig on stored limit → uplink sig on hash → bytes recompute to hash | `verifier.go` |

Note the GET `Download`, `GetChunkProof`, and `GetSignedPiece` endpoints all run the *identical*
`verifyOrderLimit` + `verifyOrder` pair, so an audit/reverify request cannot be forged or replayed any
more than a normal download can.

**CLONE root caveat (carried over from upload):** the stored CLONE `MerkleRoot` is **unsigned** — the
coordinator never signature-verified it at commit time (see
[`dataflow-upload.md`](dataflow-upload.md) §5). That is *why* a CLONE proof mismatch on intact bytes
is classified `ClientFault` rather than a worker `Failure`: the bytes are anchored by the worker's
uplink-signed piece hash, but the root might be a bad client commitment.

---

## 6. Failure & edge cases

- **Segment deleted / repaired before verify** — `GetAuditSegment` returns `vo.ErrRecordNotFound`; the
  job is dropped, not failed.
- **Expired segment in queue** — `auditOne` skips it using the denormalized expiry; no reload.
- **Worker offline / dial failure** — `classifyErr` → `Offline`; counts against the **online score**
  but is not a storage failure.
- **Worker stalls / timeout** — `Contained`; the piece is enqueued to `ReverifyQueue` rather than
  judged immediately.
- **RS: too few shares return / FEC can't decode** — responders are downgraded to **Unknown**, never a
  false pass (asymmetric: the audit refuses to credit a pass it cannot prove).
- **RS: a share disagrees with the canonical reconstruction** — that share's worker → `Failure`.
- **CLONE: proof mismatch** — `Contained` first; on reverification, intact-bytes-but-bad-proof ⇒
  `ClientFault` (file marked CORRUPTED), corrupt-bytes ⇒ `Failure`.
- **Contained too long** — fails once `ReverifyCount` hits `MaxReverifyCount`; transient
  offline/unknown retries are bounded by `maxReverifyAttempts` (10).
- **Forged / replayed audit request** — impossible: the worker runs the same coordinator-signature +
  single-use-serial checks as any download (checkpoints 1–3).
- **Process restart mid-audit** — durable DB queues (`VerifyQueue`, `ReverifyQueue`) plus
  `SKIP LOCKED` claiming mean no work is lost or double-processed.

---

## 7. Where to go next

- [`../coord/audit`](../coord/audit) — the audit subsystem README and package doc; the canonical source
  for this pipeline (`Observer`, `Producer`, `Worker`, `Verifier`, `Reporter`, `ReverifyWorker`).
- [`../coord/orders`](../coord/orders) — `CreateGetOrderLimits` and the ephemeral piece key the
  verifier signs single-use orders with.
- [`../coord/file`](../coord/file) — segments, redundancy-scheme geometry (`PieceSize`, the 4-byte
  stripe length prefix), and the stored CLONE `MerkleRoot`.
- [`../worker/piecestore`](../worker/piecestore) — the `Download`, `GetChunkProof`, and
  `GetSignedPiece` endpoints and the `verifyOrderLimit` / `verifyOrder` / `usedserials` checks workers
  run.
- [`../pkg/infectious`](../pkg/infectious) — the Reed-Solomon FEC `Correct` backing the stripe
  cross-check.
- [`../pkg/merkle`](../pkg/merkle) — `VerifyProof` / `NumLeavesFor` backing the CLONE chunk proof.
- [`../pkg/signing`](../pkg/signing) — `SignOrderLimit`, `VerifyOrderLimitSignature`, and the
  signed-piece chain used in reverification.
- [`dataflow-upload.md`](dataflow-upload.md) — the upload path that wrote the pieces and roots this
  audit reads back.
- [`../SOURCE_STRUCTURE.md`](../SOURCE_STRUCTURE.md) — the layered-peer architecture every subsystem
  follows.
