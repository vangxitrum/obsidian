---
type: fact
tags: [depin, testing, qa, test-plan]
created: 2026-08-04
agent: main
---

Testing-phase test plan written at `docs/test-plan.md` in the depin repo
(not yet committed as of 2026-08-04). Covers the 10 user-named features -
File Storage, Payment, Worker Tag, Worker Info, Usage, Reward, Register,
Deposit, Audit, Repair - plus an 11th, **Worker Auto-Update**, split out on
the user's request from Worker Info into its own section (versioncontrol
API, staged rollout cursor, `internal/version/checker`, `worker-updater`
download/stall-guard/install/restart, platform-specific restart paths).

**Format (2026-08-04, second revision):** user asked for a better format than
flat markdown checkboxes. Explored the repo first and found the team already
has a test-case convention in `.gitlab/issue_templates/{feature.md §6,
spec.md §10}`: `ID | Type | Steps | Expected` table, `TC-XX-NN` IDs, Type =
Positive/Negative-validation/Negative-auth/Boundary. No CI, no test-mgmt tool,
no other test-case format exists in the repo. Rewrote `docs/test-plan.md` to
match that convention (`ID | Type | Steps | Expected | Status` per section,
prefix per feature: FS/PAY/WTAG/WINFO/USG/RWD/REG/DEP/AUD/RPR/UPD), adding
only a `Status` column (`Not Run/Pass/Fail/Blocked`) since this doc is a
standing regression list run every cycle, not a one-time per-MR checklist.
116 rows total, 1:1 with the original checkbox lines (content unchanged,
format only). If asked again "is markdown good enough" - yes, confirmed
answer is: reuse the existing TC-ID convention, don't introduce a new tool.

**Feature -> code mapping decided (useful if the doc drifts from the repo):**
- Payment = client-side billing: `coord/wallet` (per-owner EVM deposit-address
  generation, AES-encrypted key at rest - NOT Stripe; `coord/wallet/stripe.go`
  is dead/commented-out RS-erasure "Stripe" struct, unrelated naming collision,
  user confirmed 2026-08-04 there is no Stripe/card deposit support), `cmd/coord
  billing.go` (`credit`/`balances`/`generate-invoices-csv`/`record-period`),
  `internal/billing`. Deposits are on-chain AIOZ only, see [[client-billing-phase-a]].
- Reward = worker compensation: `internal/compensation`, `cmd/coord
  compensation.go` (`generate-invoices-csv`/`record-period`/`record-one-off-payments`).
- Worker Tag = node-tag signing, see [[worker-tags-plan]] (now shipped) - NOT
  the S3-style `coord/tags` contract/file tags (that's under Usage instead,
  via `GetUsageByContractTag/FileTag`, see [[contract-file-tags-usage]]).
- Register = identity/onboarding via `cmd/keytool` (create/id/sign-ca/authorize/
  sign-server+auth/sign-tags/new-piece/keys). Worker Info = check-in/runtime
  telemetry (`worker/contact` RegisterToHub, uptime, reputation, hashstore-pieces
  debug endpoint) - these two look similar but are separate lifecycle stages
  (onboard once vs. report every check-in).
- Deposit = on-chain AIOZ deposit watcher + rate conversion + sweeper
  (`coord/deposit`), distinct from Payment/wallet even though both touch balance.

**Update 2026-08-04 (third revision):** user asked to add test cases for the
6 previously-flagged gaps too. Added sections 12-17 (OVL/RLY/GC/PLC/MON/TAG
prefixes), 45 more rows, 161 total. Researched each via codegraph_explore:
coord/overlay (UploadSelectionCache reputable/new split + distinctNet /24,
DownloadSelectionCache includes suspended unlike upload), worker/relay
bootstrap (ResolveRelayAddrs cache-fallback, OnCheckInRelayAddrs drift
warning - AutoRelay is fixed at host construction so a fleet change needs a
worker restart), coord/gc/bloomfilter+sender (Retain is idempotent, rejects
only strictly-older CreationDate), coord/placement (WorkerFilter/
UploadFilter/SelectionStrategy, ECParameters RS vs CLONE), pkg/telemetry +
pkg/logship (cumulative counters not deltas; logship self-heals by dropping
oldest on overflow), coord/tags (MaxTags=10, 128/256-byte key/value caps,
case-sensitive keys, CRLF/control-char injection guard - shared by contract
AND file tags, confirmed via `coord/storage/service.go CreateContract`).

**Update 2026-08-04 (fourth revision):** user said this doc is functional-only,
no benchmark/load-testing - removed TC-FS-14 (k6 benchmark pipeline) and
TC-FS-10 (concurrent-download hang regression, since its method was "many
concurrent downloads under load" even though its goal was correctness).
File Storage renumbered FS-01..FS-13 (was 15). 159 rows total now. Flagged to
user that dropping the concurrent-download-hang case removes doc coverage of
a real past incident (edge goroutine/memory hang) - it's not tracked here
anymore, only in the incident memory `edge-download-timeout-fix`.

New gap surfaced during this pass, not yet in the doc as a section: **worker
graceful exit** (`worker/trust/coorddb.go` CoordDB:
InitiateGracefulExit/UpdateGracefulExit/CompleteGracefulExit/
CancelGracefulExit/ListGracefulExits) - a worker leaving the network isn't
covered by any of the 17 sections. Flagged in the doc's footer, not yet
written as test cases (needs scoping first: what triggers it, in-flight
piece handling, completion receipt verification).

Side note: today's attempt to auto-dispatch this into the Obsidian vault via
`hermes chat -m kr/claude-sonnet-4.5-agentic` failed with `HTTP 404: No active
credentials for provider: kiro` - the 9router backend for that model needs
re-auth before hermes-based vault writes work again (see [[hermes-9router-provider]]
in `_global`).
