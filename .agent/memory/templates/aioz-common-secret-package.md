---
type: decision
tags: [aioz-common, security, go, secrets, api-keys]
created: 2026-09-08
agent: main
---

2026-09-08: new `aioz-common/secret` package, stdlib-only, on branch
`feat/depin-utils-and-secret`. It exists because neither aioz-common nor depin
had a shared way to mint or verify a secret, so depin grew four hand-rolled
token idioms with three storage disciplines and two comparison disciplines.
See [[aioz-common-depin-extraction]] for the branch context.

LOAD-BEARING GO FACT: since **Go 1.24, `crypto/rand.Read` cannot return a
non-nil error** - it calls `fatal()` and terminates the process (verified in
`$GOROOT/src/crypto/rand/rand.go:78`). So every generator here returns no
error, and depin's three incompatible failure policies are all dead branches on
its own Go version: propagate (`coord/admin`), return `""`
(`edgeserver/downloadid.go:30`), and fall back to `time.Now().UnixNano()`
(`coord/audit/observer.go:105`). The clock fallback is unreachable.

Design: `Token` is plaintext and redacts itself; `Hash` is what you store and
marshals freely. A secret flows in freely and out only through `Reveal()`.

TWO FMT FACTS worth keeping, both found by the tests:
- `fmt.Stringer` is consulted for `%v %s %q %x %X` **only**. `%d` takes fmt's
  badVerb path and reflects the underlying string out. Only `fmt.Formatter`
  closes every verb.
- `%p` and `%T` are handled **before** fmt consults `Formatter`, so `%p` leaks
  the value and `Format` never runs. And `go vet`'s printf check **skips any
  type implementing `fmt.Formatter`**, so the very method that closes every
  other verb is what silences vet for this one. `%p` is therefore a silent
  hole; it is asserted by a test rather than left to be rediscovered. The other
  unclosable hole is an unexported `Token` struct field, because fmt cannot
  call methods on unexported fields.

`Token.Value()` returning an error is the highest-value method: `Token`'s kind
is `string`, so without a `driver.Valuer` database/sql's default converter
reaches it by reflection and **silently stores the plaintext** - verbatim
depin's `certificate/authorization` bug.

Outbound marshallers error rather than emit the redaction placeholder: a
handler returning the literal string ships a bug that looks like a working
deploy and fails later, in another service.

`Hash` is unsalted SHA-256 hex, deliberately byte-identical to depin's
`coord/admin/credentials.go:157 HashToken`, so adopting this package there
needs no backfill. A literal golden vector pins it.

API keys are `<prefix>_<id><secret><crc>`, base62 by rejection sampling (a
naive `b%62` over the whole byte range biases the alphabet's first eight
characters; a chi-square test guards it). The embedded id is the structural fix
for `certificate/authorization/db.go:114-121`'s linear `bytes.Equal` scan. The
prefix is inside the hashed material, so it doubles as the scheme version with
zero bytes added. The CRC32 is explicitly not a MAC.

Recovery codes are Crockford base32, not RFC 4648. depin's
`credentials.go:224` claims base32 has no 0/O or 1/I ambiguity; that holds in
one direction only. RFC 4648 offers **no input folding**, so a person who reads
`O` and types `0` is rejected at the exact moment they have already lost their
phone. Ambiguity is a parsing property, not an alphabet property.

**Why:** one audited package makes a future security fix or CVE response a
version bump instead of a repo-by-repo hunt.

**How to apply:** migrating depin's `coord/admin` is a drop-in with no data
change. `certificate/authorization` fixes plaintext-at-rest, the
non-constant-time compare and the O(n) scan at once, but the presented format
changes, so unclaimed tokens are invalidated. Recovery codes are **not** a
drop-in: existing codes were minted from `A-Z2-7` and can contain literal `I`,
`L`, `O`, which Crockford folds to `1`, `1`, `0`, changing the hashed string -
re-issue at next MFA enrolment rather than carrying a dual-verify path.

The highest-severity audit finding is NOT fixed by this package - it needed
the configuration layer instead. See [[aioz-common-config-secret-guard]].
