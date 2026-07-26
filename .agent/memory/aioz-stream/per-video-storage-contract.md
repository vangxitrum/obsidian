---
type: fact
tags: [go-sdk, storage, contracts, accounting, proto, depin]
created: 2026-07-16
agent: main
---

Per-video tagged storage contracts in aioz-stream, threaded into the RegisterJob payload
(go-sdk `37bb2ea`). Left **uncommitted** in the working tree - see "Commit hygiene" below.

**2026-07-17 update - user changed the fallback + contract now minted at media creation.**
User edited `newUploadParams` (gosdk.go) so an empty `contractID` is now a **hard error**
(`"gosdk: contract id is empty"`) instead of falling back to a default contract, AND removed
the boot-time default contract + the `contractID` field from `GoSdkHelper`. Intentional
(flagged as such by the harness). Consequence: every upload MUST carry a non-empty contract.
This broke thumbnail/caption/chapter uploads (media had empty `contract_id`) and the 3
user-level sites that passed `""` (watermark, player-theme logo, playlist thumbnail).
Fix (user chose "mint at media creation"): mint the contract in `createMediaObject`
(video.service.go, the single choke point for both `CreateMediaObject` and
`CreateMediaObjectLiveStream`) **fatally** before `mediaRepo.Create` - no billing yet there,
and a contract-less media is unusable, so fail fast. Removed the (non-fatal) mint block from
`completeMedia`; kept `ContractId: media.ContractId` in the RegisterJob payload. The 3
user-level sites now mint their own contract inline tagged `user_id` (+ asset id). Builds +
vet green.

**BLOCKER (environmental, not code) - the local go-sdk identity is UNREGISTERED on the
deployed coord again.** `CreateContract` now fails: `sdk: create contract: rpc error: code =
NotFound desc = storage: not found`. Traced to coord source (depin `coord/storage/service.go:8-11`):
`clientRepo.GetByID(ownerID)` returns `vo.ErrRecordNotFound` -> `StorageError.Wrap(vo.NotFound)`
= "storage: not found" = the **client (owner) isn't registered**, NOT a placement problem
(placement errors say "unknown placement_id"). `GetAccount` also returns `NotFound: client:
not found`. The identity files (`secrets/gosdk-identity/uplink/`) are unchanged since Jul 10,
so the identity didn't change - **the coord was redeployed/reset** (same coord that started
returning `Unimplemented` for `GetUsageByContractTag`), wiping the 2026-07-13 `RegisterAccount`
(client id `bb010caa-...`). This is ALSO the real reason all 29 existing media have empty
`contract_id`: the old non-fatal completeMedia mint was silently failing with this exact error.
**To unblock: re-register via `client.RegisterAccount(ctx, email)` (needs user OK - live
network transaction) OR point at a coord where this identity is registered.** With the new
fatal mint, media creation itself now fails until this is resolved - surfaces the real error
instead of the old silent-empty-contract -> confusing "contract id is empty" at upload.

**RESOLVED 2026-07-17 (user-confirmed):** re-registered via `RegisterAccount(ctx,
"premium.accounts@aioz.io")` against `68.183.189.51`. Returned the SAME client id as
2026-07-13 (`bb010caa-7996-63de-3822-9d9800548d01`) - re-registering the same public key
restores the same record. Full chain then verified live end-to-end (throwaway, deleted after):
GetAccount NotFound -> RegisterAccount OK -> GetAccount returns the record -> `CreateContract`
(the media-creation path) succeeds (`019f6e0a-...`) -> thumbnail-style `Uploads` under that
contract uploads 2 objects OK -> empty contract still rejected. So the coord loses
registrations on redeploy; if `CreateContract`/`GetAccount` start returning `NotFound: client:
not found` again, just re-register (same client id comes back). **Note: the running local
`api` binary (started Jul 16) predates this code + must be rebuilt/restarted to pick up the
fatal-mint-at-creation change; and existing media (all deleted, empty contract) can't be used
to test thumbnail upload - create a NEW media, which now gets a contract at creation.**

**All 29 media of the only DB user (`tuantq666@gmail.com`, `c86216a4-...`) set to status
`deleted`** on 2026-07-17 at user request (was 12 done / 3 fail / 1 deleting / 13 deleted).
Note: no `premium.accounts@aioz.io` user exists in this local `video-db`.

**go-sdk `37bb2ea` was already local HEAD** when the user asked to "use 37bb2eae8c41" -
the `replace aioz-depin/go-sdk => /home/tuan/work/depin-workspace/go-sdk` makes version
bumps invisible. "Use version X" here means *adapt to its API*, not edit go.mod. Confirms
the takeaway already in [[gosdk-storagehelper]]: always `git log -1` in go-sdk first.

**The breaking change:** `3f950cc` ("feat(accounting): add contract tags and usage query
by tag") gave `CreateContractWithPlacement` a 4th param `tags map[string]string`, breaking
the build in *both* aioz-stream (`internal/utils/storage/gosdk.go:67`) and aioz-stream-core
(`internal/utils/cdn/gosdk.go:72`) - same error, both repos. `37bb2ea` itself is only error-string
churn (`"uplinksdk: "` -> `"sdk: "`), but it does change the *value* of exported
`ErrNotImplemented` - anything asserting on `err.Error()` text breaks.

**Why per-video contracts at all:** tags attach to *contracts*, not files, are immutable
once set, and are readable only via `GetUsageByContractTag(ctx, tagKey, tagValue, from, to)`
(account-scoped, no contract-id param). So per-video usage attribution *requires* per-video
contract granularity. Both repos previously created ONE contract at boot and reused it.

**Silent-failure trap (the whole reason this needs an attribution test):** a zero
`UploadParams.ContractID` makes go-sdk mint a random client-side UUID (`upload.go:70-73`),
never return it, and carry no tags -> usage permanently unattributable. Coord also accepts
contract IDs it never issued. Every failure mode degrades to "works fine, unattributed" -
build/vet/tests prove nothing here.

**Implementation shape** (all in aioz-stream; core deferred by user choice):
- `StorageHelper` gained `CreateContract(ctx, tags) (string, error)` + a trailing
  `contractID string` on all 5 upload methods (mirrors the earlier `systemHeaders`
  threading). `GoSdkHelper.newUploadParams` substitutes `h.contractID` when empty - one
  choke point, all 5 methods route through it. `CdnHelper` ignores the param and returns
  `("", nil)` from `CreateContract` - **not** an error, or every upload on that backend fails.
- Rejected: smuggling the contract via `systemHeaders` under a reserved key, or a context
  value. Both fail *silently* at a missed call site; an explicit param makes the compiler
  enumerate all 9. This was the right call - the compiler found every site immediately.
- `models.Media.ContractId string` next to `JobId` (gorm AutoMigrate, no migration file).
- Contract minted in `completeMedia` (`video.service.go`, just before the
  `RegisterJobRequest` literal), tags `{"video_id","user_id"}`.
- `string contract_id = 5;` on `RegisterJobRequest`; field 5 free in both repos' proto copies.

**Two non-obvious correctness points found by reading, not by testing:**
1. **Contract-create errors MUST be non-fatal.** The user is billed at `video.service.go:3206`
   and the ONLY refund in the file is at `:3974`, inside `handlePlaylist`'s `FailStatus`
   branch - which runs only *after* `RegisterJob` succeeds. Any `return err` between those
   points rolls back the media but leaves the charge standing. Log + fall back to the
   default contract instead. (A Plan agent recommended "fatal, matching precedent" here -
   that would have shipped a money-loss bug.)
2. **Idempotency guard `if media.ContractId == ""` is required.** `HandleLiveStreamMedias`
   calls `completeMedia` in a cron loop with no transaction and only logs errors, so it
   retries every tick and would otherwise mint a duplicate contract per pass.

**BLOCKER - attribution is unverifiable against the live coord.** `68.183.189.51:7777`
returns `Unimplemented: unknown method GetUsageByContractTag for service
hub.accounting.v1.AccountingService`. The coord *source* (depin repo, HEAD `3813b19`)
does implement it (`coord/accounting/usage.go:70`) and does persist tags
(`coord/storage/service.go:68`, `coord/storage/contract.go:10`) - the deployed build is
just stale. Since it predates the feature, it almost certainly **silently dropped** the
tags on my test contract (proto3 unknown-field). Live-verified as working anyway:
contract creation with tags accepted, upload under that contract, and `GetLink` all
succeeded. **Re-run the attribution check once coord is redeployed from depin main.**

**Coord tag validation** (`depin/coord/tags/tags.go`): max 10 tags, key <=128 bytes,
value <=256 bytes, printable ASCII only (rejects C0 controls + DEL, CRLF-injection guard).
Keys are case-sensitive and NOT lowercased. `video_id`/`user_id` -> UUID strings comply.

**protoc regen without sudo** (protoc isn't installed; apt needs sudo). Working recipe:
download `protoc-21.12-linux-x86_64.zip` from GitHub releases into a scratchpad + `go install
google.golang.org/protobuf/cmd/protoc-gen-go@v1.36.1` - those exact versions match the
header in the committed `job.pb.go` (`protoc-gen-go v1.36.1` / `protoc v3.21.12`), giving a
diff confined to the new field + rawDesc. Using `buf` (installed) or protoc-gen-go v1.36.11
instead "works" but emits the newer codegen style (rawDesc as string, `unsafe` import,
`protoc (unknown)` header) = 632-line diff. Also: `buf` chokes on this repo's
`option go_package = "/job"` leading slash unless you pass `paths=source_relative`.

**Gotchas re-confirmed:** build with `go build -tags purego ./cmd/... ./pkg/... ./internal/...` -
`go list ./...` dies on `open postgres_data: permission denied` (not just the `seeds` break
noted in [[gosdk-storagehelper]]). The Slack `AuthTest()` boot blocker from [[local-dev-env]]
is **gone** - commented out at `internal/utils/message/message.go:28`; that memory is stale.

**Commit hygiene - real mistake made here.** ~10 of the 13 files this change touches already
had uncommitted edits from the user's concurrent session (the pattern [[gosdk-storagehelper]]
already warns about). `git add <file>` stages the *whole file*, so an initial "just the
build fix" commit silently swallowed 86 lines of the user's default-contract work under a
misleading message. Soft-reset at the user's direction; all work left uncommitted for them
to separate. **On this repo, assume every file you touch has someone else's uncommitted work
in it - check `git status` before `git add`, and prefer `git add -p` or leave it to the user.**
