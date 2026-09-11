---
type: decision
tags: [aioz-common, postgres, redis, rabbitmq, httpclient, ssrf, config, migration]
created: 2026-09-10
agent: main
---

We moved Postgres, Redis, RabbitMQ and outbound HTTP onto aioz-common. The plan is `Projects/aioz-stream/plans/2026-09-10-aioz-common-v030-clients.md`, with a "Revision 2026-09-11" section at the end. As of 2026-09-11 it is **implemented and verified on branch `refactor/sync-with-template-source`, but not yet committed**.

**The pin:** aioz-common commit `5343511` (2026-09-11), which comes after `6b79f13` on `feat/http-postgres-retry`. It is v0.3.0 plus the SSRF guard, with retry moved from the transport into a caller-driven policy. It is untagged.
- The resulting pseudo-version is `v0.2.1-0.20260911043352-53435114951d`. Its base is v0.2.0, because the v0.3.0 tag sits on a merge commit that is not an ancestor.
- That pseudo-version sorts below v0.3.0. If any dependency requires ≥ v0.3.0, MVS drops these commits.
- Re-pin to v0.3.1 once it is tagged.

**Retry at 5343511:**
- `WithRetry`/`WithoutRetry` and the `Retry*` Config fields are gone, and the client never retries on its own.
- A call site now retries with `httpclient.Retry{Name, MaxAttempts, Retryable, Wait, Log}.Do(ctx, client, newRequest)`.
- `newRequest` runs once per attempt, so headers have to be set inside it.
- Each attempt gets its own `Client.Timeout`. The caller's ctx deadline bounds the whole sequence; a wait that cannot finish before the deadline is not slept.
- `Retry.Do` sees http.Client errors, including CheckRedirect refusals, which the old transport retry never saw. `DefaultRetryable` treats a url.Error as transient, so hlsimport uses a custom `Retryable` that stops on `ErrHostNotAllowed`, `ErrTooManyHops` and `ErrInvalidScheme`.
- In aioz-stream, each client holds a `retry` field (stream, transcribe, summary, ip2location, hlsimport). Stream call sites use the `c.request(method, target)` builder.

**How the httpclient guard behaves:**
- `NewOneShot` blocks internal addresses by default. `New` allows them unless the config sets `private-networks: block`.
- `NewWith` is guarded only when its transport came from `Transport` or `OneShotTransport`.
- A refusal wraps `ErrBlockedAddress` and is never retried.
- `resty.NewWithClient` keeps both the Transport and httpclient's CheckRedirect.

**User decisions (2026-09-10 and 2026-09-11):**
- Config: adopt the common Config structs as config sections.
- Queues: keep them non-durable and non-persistent.
- Webhook: `httpclient.New` with `block` as the default. A release build refuses `allow`. `Check` returns 422 `webhook-url-not-public`, and `Notify` does not queue a retry for a blocked address.
- hlsimport: swap to the common guard, with `CheckRedirect` re-set so `CheckURL` still runs.
- stream: `GetStreamPathWithStreamType` keeps its own retry loop, because it also retries a 200 with an empty path.
- cdnHandlerCh, qualityCh and responseCh: dropped entirely.
- redis.conf: edited to `bind * -::*` and `protected-mode no`, then loaded through the compose `command:`. The stock file binds loopback only, which cuts off the api and nginx.

**Traps:**
- `postgres.Config` never sets the session TimeZone. We set `RuntimeParams["timezone"]` after `PoolConfig()`, and three queries depend on it.
- The release default for `ssl-mode` is `require`.
- `httpclient.NewWith` overwrites CheckRedirect.
- The 32MiB response cap: it is off for cdn and hlsimport.
- The `default:` tags apply only through `BindFlags`. We run it on a throwaway flagset, then `structs.Decode` the file section.
- The rabbitmq package has no disk spool (dropped), and `rabbitmq.Consumer` has no `Name()`.
- The gorm 1.25 logger has no slog adapter, so `internal/config/gormlog.go` provides one.
- The stream path constants have no leading slash, because `STREAM_API_URL` ends in `/v3/`. Tests must use that same URL shape, or they fail on a URL parse error and look like a pass.

**Verification gotchas:**
- The user's main checkout (`~/work/stream/aioz-stream`) runs an old `./bin/api` that consumes callWebhook, endLiveStream and the highlight queues on the shared dev broker `:5675`. It steals broker E2E messages. Run broker tests against a throwaway `rabbitmq:3-management` container instead. In a fresh container `rabbitmqctl await_startup` never answers, so wait for `docker logs | grep 'Server startup complete'`.
- `TestMigrationsApplyToAnEmptyDatabase` runs `Up` on `MIGRATE_TEST_DSN`, so point it at an empty scratch database.
- The highlight integration test needs `HIGHLIGHT_IT_USER_ID` set to an existing user, because of the editor_projects→users foreign key.
- Edit existing Go files with Python, not the Edit tool: [[go-edit-formatter-hook]].

**Still open (ops):**
- Rewrite each host's config.yaml before deploying: `postgres.database`, `ssl-mode` and the pool, `redis_conn_id`/`redis_uuid`, `rabbitmq.address`/`username`/`password`, and `http.webhook`.
- `storage.recover_path` is now unused.
- Queues become durable only in coordination with core and mediamtx. RabbitMQ 4 forces this change ([[rabbitmq4-transient-queues]]).
- Redis is published on 6379 without a password, which is the same as before.

Related: [[legacy-tables-drop-candidates]], [[template-alignment]], [[monkit-observability-migration]], [[aioz-common-uuid-text-columns]], [[single-binary-cli]].
