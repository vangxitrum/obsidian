---
type: fact
tags: [storj, docs, architecture]
created: 2026-07-29
agent: main
---

The `storj/storj` repo root has a growing series of deep-dive architecture
docs, one file per subsystem, written in a dense, file:line-referenced style
(numbers first, code snippets, tables, a "mental model" ASCII diagram, and a
"Key files" table at the end). Examples: `audit.md`, `audit-window.md`,
`repair.md`, `reputation.md`, `overlay.md`, `node-selection.md`,
`upload-selection.md`, `download-selection.md`, `durability.md`, `encrypt.md`,
`ip-resolve.md`, `planet.md`, `reach.md`, `record.md`, `tls.md`, `autoupdate.md`,
`cmd.md`, `research.md`.

`reward.md` was added 2026-07-29, covering storage node reward/payout end to
end: `satellite/compensation` (rates, withholding schedule, `GenerateStatements`
money math), how usage is metered (`satellite/accounting/nodetally` for at-rest
byte-hours, `satellite/orders/endpoint.go` settlement for bandwidth,
`satellite/accounting/rollup` merging both into `accounting_rollups`), the
`cmd/satellite/compensation.go` operator CLI (generate-invoices-csv /
record-period / record-one-off-payments), and the node-facing views
(`satellite/snopayouts` served over DRPC vs. `storagenode/payouts/estimatedpayouts`
computed live and locally, mirroring the same held-rate schedule client-side).
Note: there is no node-initiated "withdraw" flow in the repo at all - payout
is a satellite-side push (external `crypthopper-go` batch tool reads the
invoice CSV and transfers tokens), gated by a minimum-payout threshold, to a
statically-configured wallet address; asked to remove that section from the
doc afterward, so it's not in `reward.md` itself but worth knowing if asked again.

`client-usage.md` was added 2026-07-30, the mirror-image doc: how Storj meters
**project/customer** usage (as opposed to node reward) for live limit
enforcement and Stripe billing. Key asymmetry vs `reward.md`: client billing
uses `total_encrypted_size` (logical, pre-erasure-expansion bytes from
`satellite/metabase/accounting.go`), while node payout uses physical
per-piece byte-hours (erasure-expanded) from `nodetally`. Two layers: live
Redis counters (`satellite/accounting/live`, fail-open gate for
upload/download limits) vs. durable `bucket_storage_tallies` +
`bucket_bandwidth_rollups` (billing source of truth, integrated as a
byte-hour step function in `satellite/satellitedb/projectaccounting.go:
GetProjectTotalByPlacement`), feeding `satellite/payments/stripe/service.go`
invoicing.

**Why this matters:** when asked to explain or extend a Storj subsystem, check
for an existing root-level `<topic>.md` first — it may already contain the
file:line map at the right depth, and new docs should match this style/format.

**How to apply:** before researching a subsystem from scratch, `ls *.md` at the
repo root; if a matching doc exists, read it first instead of re-deriving.
