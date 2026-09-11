---
type: decision
tags: [aioz-common, jwt, security, go]
created: 2026-09-08
agent: main
---

2026-09-08, branch `feat/depin-utils-and-secret`: `aioz-common/jwt` added at
the user's explicit request, after I recommended against it. Both signing
styles: HMAC shared secret (HS256/384/512) and key pair (RS*, PS*, ES*,
EdDSA). Dependency: `golang-jwt/jwt/v5` v5.3.1, which has NO dependencies of
its own (one module, 42 -> 43).

WHEN TO USE IT AT ALL, and the argument I made first: a JWT is self-describing
so the verifier trusts claims without a lookup, which is exactly why it cannot
be revoked before it expires. It only pays off when the verifier CANNOT reach
the issuer's database (third party, edge worker, another company). For coord
and edgeserver, which already talk to the coordinator every request, an opaque
[[aioz-common-secret-package]] token is strictly better. This is written into
the package doc and README so the next person meets the argument.

THE ATTACK the design is built around, verified working: given a token signed
RS256, an attacker re-signs their own claims with HS256 using the RSA PUBLIC
key (x509.MarshalPKIXPublicKey bytes) as the HMAC secret. A verifier whose
keyfunc switches on `token.Method` and returns the public key for HMAC accepts
it - I PROVED this in a scratch module: "NAIVE VERIFIER ACCEPTED THE FORGERY,
sub=admin valid=true". Defence: `gojwt.WithValidMethods([]string{alg})` from a
constructor-pinned algorithm, plus a second explicit header check inside the
keyfunc (deliberately redundant - that callback is where every published
alg-confusion bug was written, so a reader should find the check there).

Structural defences chosen over documentation:
- exp required on BOTH sign and verify; opt-outs are named
  `AllowTokensThatNeverExpire` / `AcceptTokensThatNeverExpire` so they are
  visible in review and unreachable via a zero value.
- HMAC secret must be >= hash size (32/48/64 bytes). Guessed offline from one
  captured token, no rate limit. My own example hit this: `secret.Bytes(32)`
  with HS512 returned an error I ignored, then nil-deref'd.
- Key type checked in the constructor (Ed25519 key into an RS256 verifier =
  startup failure, not a 500).
- `Extra` claims cannot shadow iss/sub/aud/exp/nbf/iat/jti.
- `Sign` returns a `secret.Token`, so a JWT inherits the redaction.

`Ring` does kid-based rotation. It REQUIRES a kid on every member rather than
falling back to trying all keys: a try-them-all ring accepts a token signed by
any key it ever knew and hides that the old key was never retired. Reading the
kid uses `ParseUnverified`, which is safe only because a kid routes among keys
already trusted - a test asserts that naming a known kid does not skip the
signature check.

Both `Sign` and `Verify` take a ctx and read the clock via
`time2.Now(ctx)` (golang-jwt's `WithTimeFunc`), so expiry tests advance a
`time2.Machine` instead of sleeping. Good cohesion argument for having ported
time2 in [[aioz-common-depin-extraction]].

Out of scope by decision: JWKS, refresh flows, revocation lists, JWE.

Coverage 88.7%, race-clean. See also [[aioz-common-version-build-stamp]] for
the other package added the same day.
