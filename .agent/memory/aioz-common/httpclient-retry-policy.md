---
type: decision
tags: [httpclient, retry, api]
created: 2026-09-11
agent: main
---

httpclient no longer retries inside the client. The retry RoundTripper, `WithRetry`/`WithoutRetry` and the `Retry*` Config fields were removed (branch `feat/http-postgres-retry`, uncommitted as of 2026-09-11). Retrying is now explicit at the call site:

```go
httpclient.Retry{Name, MaxAttempts, Retryable, Wait, AllowNonIdempotent, Log}.Do(ctx, client, newRequest)
```

- `Retryable func(resp *http.Response, err error) bool` - whether to retry. `DefaultRetryable` = network errors (not TLS / ErrBlockedAddress / redirect refusal), 429, 5xx except 501, and gives up on Retry-After > `MaxRetryAfter` (30s). `RetryableWithin(limit)` changes that cap.
- `Wait func(attempt int, resp, err) time.Duration` - pure duration, no Stop sentinel (the user wanted each func to have one job). Built-ins: `Exponential(min,max)` (full jitter), `Constant(d)`, `RetryAfter(fallback)`. `ParseRetryAfter` exported.
- POST/PATCH without Idempotency-Key get one attempt unless `AllowNonIdempotent`.

**Why:** the user judged the context-flag API confusing: silent no-ops (WithRetry on POST, non-rewindable body), and the flag leaked to every downstream call via context inheritance. The aioz-template reference even wrapped a POST in a no-op WithoutRetry. User's view: "retry is the developer's job, not the package's".

**How to apply:** aioz-template's `rate` service already uses `Retry.Do`. Docs were updated in a follow-up the same day: httpclient/README.md ("Retrying is the caller's job" section + API block), docs/ADOPTING.md, a CHANGELOG entry marked breaking, labelled v0.3.1 at the user's request (semver says v0.4.0; see [[v0.3.1-release]]), and aioz-template rate/README.md. The v0.3.0 release note still mentions WithRetry on purpose - it is history. Related: [[httpclient-ssrf-guard]], [[v0.3.0-release]].

- 2026-09-11 review (v0.3.0..HEAD): Retry.Do checks Idempotency-Key only on attempt 0 and callers build it inside newRequest, so each retry can carry a fresh key (double POST risk); guard allowlist not matched against NAT64/6to4-embedded IPv4 (fails closed). Build/vet/tests pass.
