---
type: decision
tags: [cli, cobra, config, deploy, aioz-stream]
created: 2026-09-10
agent: main
---

On branch `refactor/sync-with-template-source`, commit "refactor(cli): merge binaries into aiozstream": the three binaries (`cmd/http`, `cmd/grpc`, `cmd/migrate`) became one, `cmd/aiozstream`, with the subcommands `api`, `grpc`, `migrate up|down|version|force` and `version`. (2026-09-10: the migrate subcommand was removed; the upstream golang-migrate CLI runs migrations instead. See [[upstream-golang-migrate]].) The entry points live in `internal/cli/{api,grpcserver,migrate}`. Shared flag state is in `internal/cli/config.go` (`Globals`, `ConfigPath()`).

**Config selection:** a global `--config-dir` flag (default `.`), and the file read is always `<config-dir>/config.yaml`. `APP_ENV` was removed on the user's decision. It had different defaults per entry point (debug vs app). `debug.yaml` became `config.yaml` locally, and `env-example/app.yaml` became `env-example/config.yaml`. The livestream (mediamtx) compose services keep `APP_ENV` and the `/app/app.yaml` target: the submodule wasn't checked out, so what it reads couldn't be verified.

**Why plain PersistentFlags, not depin's cfgstruct.SetupFlag:** depin scans `os.Args` early only because `Bind` bakes `$CONFDIR` into struct defaults before parsing. Here config loads inside Run, after cobra has parsed.

**Deploy prerequisite (ops, not yet done as of 2026-09-10):** on each host, in the compose dir (`dirname $DEPLOY_PATH`), run `mv app.yaml config.yaml` and upload the new compose file. CI `migrate-job` runs `/tmp/aiozstream-migrate --config-dir <compose dir>` and fails early if `config.yaml` is missing.

**Gotchas hit:**
- `aiozstream version` printed to stderr (cobra `cmd.Print`), so `make verify-stamp` failed on every build. See [[cobra-print-goes-to-stderr]].
- In this treehouse worktree, `go build` records no VCS block, even with `-buildvcs=true`: `.git` points into another checkout's worktrees dir. The stamp shows a zero timestamp locally. CI clones are fine.
- A local api/grpc boot panics in `storage.MustNewCdnHelper` (it makes an eager `GET <cdn_url>/getBalance`) with the placeholder CDN URL. For e2e, run a stub returning `{"deposit_address": <chain.business_address>, "set_credit_later": true}` and set `CDN_URL=http://127.0.0.1:<port>`.
- The user's main checkout (`~/work/stream/aioz-stream`) runs `./bin/api` on :3000 and :6060, which gave a phantom `pong`. Before trusting a response, confirm port ownership with `ss -ltnpH 'sport = :N' | grep pid=$P`, and move with `PORT=` / `DEBUG_ADDR=`.

Related: [[template-alignment]], [[local-dev-env]], [[repo-layout]].
