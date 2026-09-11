---
type: project
tags: [aioz-common, httpclient, security, ssrf]
created: 2026-09-10
agent: main
---

# httpclient private-network (SSRF) guard - design decisions

Added 2026-09-10 (uncommitted at time of writing), folded into the untagged v0.3.0.

- Check is in `net.Dialer.Control` on the resolved IP, never on the URL host: defeats
  DNS rebinding, and every redirect hop is a new dial so redirects need no extra policy.
- `PrivateNetworks` is tri-state: `""` lets the constructor decide (`New` allows,
  `NewOneShot` blocks - one-shot is the client for user/partner-supplied URLs), or
  `allow` / `block`. Setting `AllowedNetworks` turns blocking on even for `New`; only
  `allow` switches the guard off.
- A guarded transport sets `Proxy: nil` - through a proxy the guard only sees the proxy.
- Blocked: v4 private/loopback/link-local/CGNAT/reserved/doc/multicast, v6 ULA/link-
  local/site-local/multicast/Teredo/doc/discard, and internal v4 embedded in v4-mapped,
  NAT64 (64:ff9b::/96) and 6to4 (2002::/16). Zones stripped first: `netip.Prefix.Contains`
  is false for a zoned address.
- `ErrBlockedAddress` is non-retryable in `retryableError`; metric `http_dial_refused`
  tagged name+reason. `NewWith` with a caller-built transport is only guarded if it came
  from `Transport`/`OneShotTransport`.
- Behavior change: `NewOneShot` to an internal address now fails; fix with
  `allowed-networks` or `private-networks: allow`.
