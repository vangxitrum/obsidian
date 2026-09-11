# aioz-common — memory index

Shared Go library module for AIOZ services (`gitlab.internal/tuan.quang.tran/aioz-common`).

- [[redis-and-rabbitmq-packages]] — the two client packages added 2026-09-09, what each deliberately does and does not own.
- [[amqp091-confirm-close-is-nack]] - amqp091 settles pending deferred confirms as (false, nil) on channel close, so Publisher reports a dropped connection as a broker NACK and skips its retry; plus other whole-module review findings (lrucache Add leak, memory.Size.Set panic, consumer drain-timeout dead-lettering).
- [[v0.3.0-release]] - v0.3.0 release note covers whole history; whole-module review verdict fix-first (17 findings, 5 major); note/review paths.
- [[httpclient-ssrf-guard]] - httpclient SSRF guard: dial-time resolved-IP check, tri-state private-networks (NewOneShot blocks by default), proxy off when guarded.
- [[httpclient-retry-policy]] - httpclient retry moved out of the client into explicit `Retry.Do` with `Retryable(resp, err) bool` + `Wait(attempt, resp, err) Duration`; WithRetry/Retry* config removed; docs + CHANGELOG (Unreleased, breaking) updated.
- [[v0.3.1-release]] - v0.3.1 note (SSRF guard + caller-driven retry), review fix-first (Idempotency-Key per attempt, NAT64 allowlist, patch-vs-minor); v0.3.0 tag is a merge on main so describe returns v0.2.0 - diff v0.3.0..HEAD.
