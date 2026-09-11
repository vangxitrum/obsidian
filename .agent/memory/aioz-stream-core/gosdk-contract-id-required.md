---
type: fact
tags: [gosdk, depin, upload, contract, coordinator, bug-fix]
created: 2026-08-27
agent: main
---

A job registered with an empty `contract_id` fails every upload on the go-sdk backend
with `gosdk: uploads: segment_0NN.ts: sdk: create file: rpc error: code = NotFound
desc = file: not found`, only after the whole transcode has run.

**Chain:** empty `jobs.contract_id` -> `job.ContractId` "" -> `newUploadParams` left
`params.ContractID` at the zero value -> go-sdk `UploadFile` does
`if contractID.IsNil() { contractID = vo.MustNewID() }` (`go-sdk/upload.go`), a fresh
local UUIDv7 that was never registered anywhere -> coordinator
`coord/file.Service.Create` does `s.contracts.GetByID(ctx, contractID)`, gets
`vo.ErrRecordNotFound`, returns `FileError.Wrap(vo.NotFound)` = the "file: not found"
text. **Empty contract id can never work on this backend** - the earlier comments in
`gosdk.go`/`cdn.go` claiming go-sdk "auto-generates a fresh ad-hoc contract" were wrong
and have been corrected.

The segment named in the error is arbitrary: `Uploads()` ranges a Go map, so all
segments fail and the message names whichever came first (saw segment_002 and
segment_022 on the same job).

**Fix in this repo:** `cdn.ErrMissingContractID` plus a guard at the top of
`GoSdkHelper.Upload` / `Uploads` / `newUploadParams` (so `UploadRaw`/`UploadZip` inherit
it) - fails in ~1s with a named cause instead of after the transcode. Covered by
`internal/utils/cdn/gosdk_test.go`. It cannot be a registration-time requirement because
the legacy `CdnHelper` backend ignores `contractID` entirely.

**Real repair is upstream:** the caller of the register-job API must pass a contract id
minted by `go-sdk`'s `CreateContractWithPlacement`. Nothing in stream-core ever creates
a contract and there is no config knob for one. Two constraints on that contract:
- it must already exist on the coordinator;
- it must be owned by the identity the worker runs as (`GOSDK_IDENTITY_DIR`) - coord
  checks `contract.OwnerId.Equal(peerIden.ID)` and returns `Forbidden`, a *different*
  error, on mismatch.

**Debug recipe:** `docker exec job-db psql -U admin -d w3streamcore_job -c "select id,
contract_id, status from jobs where id='<jobId>'"` (creds in `server.env`; the container
is `job-db`, not `aioz-stream-db`). Related: [[merge-new-cdn-update-transcode]],
[[gosdk-system-headers-upload]].
