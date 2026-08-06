---
type: fact
tags: [depin, worker-tags, node-tags, nodetag, storj-port, shipped]
created: 2026-07-22
agent: main
---

**UPDATE 2026-08-04**: shipped. Branch `feat/worker-tags` is merged into
`develop` (same HEAD commit `702ee23`, confirmed via
`git merge-base --is-ancestor`). Foundation-first scope below was built as
planned: `pkg/nodetag` (Authority.Verify/LoadAuthorities/Include),
`coord/contact/worker_tag.go` + `worker_tags_debug.go`
(`GET /worker-tags/?worker_id=` → tags + `is_trusted`), `coord/db/worker_tags_repo.go`,
`worker/contact/tags.go`, `cmd/keytool/cmd_sign_tags.go` (`keytool sign-tags
[authority]`). `Worker.IsTrusted` is set only from an authority-signed
`trusted_node=true` tag, never a self-signed one. Placement/selection
consumption is still deferred (Phase 2, per plan) — confirm before assuming
tags affect upload/download selection. Referenced from
`docs/test-plan.md` (repo) / `[[depin-test-plan]]` Section 3, which treats
this as "Worker Tag" distinct from the older S3-style `coord/tags`
(contract/file tags, see `[[contract-file-tags-usage]]`).

Original plan (2026-07-22) below, kept for the design rationale.

Plan to port Storj **node tags** → depin **worker tags** (signed key=value pairs
a worker advertises on check-in). Plan note:
`Projects/depin/plans/2026-07-22-worker-tags-port.md` (also in `[[hub-overview]]`
project). Source doc: `storj/worker-tags.md`.

**Decisions (from the user):**
- **Foundation first**: worker advertises → coord verifies + stores in a signed
  `worker_tags` table → tags **readable** via a coord `debug.Extension`
  (`/worker-tags/?worker_id=`). Placement/selection *consumption* (a typed
  `TagFilter` + `SelectedWorker.Tags` loading + selection caches) is **deferred to
  Phase 2** — keeps the selection hot path untouched in v1.
- **Full signing parity**: self-signed (worker key) **+** authority-signed (coord
  `TagAuthorities` trusted-cert config + a new `keytool sign-tags` CLI), incl. the
  special `trusted_node`→`workers.is_trusted`.

**Why the port is clean:** depin `pkg/signing`/`pkg/identity`/`pkg/pkcrypto` are
near-verbatim ports of `storj.io/common`, so `shared/nodetag` → new `pkg/nodetag`
maps 1:1 (`SignerFromFullIdentity`/`SigneeFromPeerIdentity`). Worker
`WorkerInfo.ID == peer.Identity.ID`, so self-signed `node_id` binds to the
check-in `id`.

**Key touch points:** proto `pkg/pb/shared/nodetag/v1/nodetag.proto` (new) +
`CheckInRequest.signed_tags`; worker `worker/contact/{tags.go(new),service.go}` +
`worker/peer.go:350`; coord `coord/contact/{service,endpoint,worker,store}.go` +
`setupContact` (`coord/peer.go:285`); DB migration `000033_worker_tags` +
`coord/db/worker_tags_repo.go`. Existing S3 `coord/tags` pkg is UNRELATED (naming
collision). Not yet implemented — plan only.
