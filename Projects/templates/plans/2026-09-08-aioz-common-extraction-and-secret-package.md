# aioz-common: extract depin's generic utilities + add a `secret` package

## Context

`aioz-common` (`gitlab.internal/tuan.quang.tran/aioz-common`, at
`/home/tuan/work/templates/aioz-common`, clean tree, tagged `v0.2.0`) was
assembled from `aioz-config`, `aioz-logger`, `aioz-stats` and
`aioz-template/internal/utils`. It holds `apierr`, `config`, `currency`,
`cycle`, `lifecycle`, `logger`, `logging`, `uuid` and
`stats/{debug,monitor,version,testcontext}`.

`depin` (`/home/tuan/work/depin-workspace/depin`, module `aioz-depin`) does not
import `aioz-common` at all today. It carries roughly 2,165 LOC of genuinely
generic, domain-free utility code that every AIOZ Go service otherwise
re-implements. Moving the clean subset up means one copy to maintain and one
place to patch.

The second half is security. An audit of both repos found **no shared way to
mint or verify a secret** - `aioz-common` has no `crypto/rand`, no hashing and
no token helper at all, and `depin` has four independent hand-rolled token
idioms with three different storage disciplines and two different comparison
disciplines. That divergence is already producing real weaknesses:

- `certificate/authorization/authorizations.go:80` mints a 64-byte
  cert-signing bearer token, stores it **verbatim (unhashed)** in a bolt
  bucket, and verifies it with `bytes.Equal` (`authorizations.go:138`) inside a
  **linear scan** (`certificate/authorization/db.go:114`) - a non-constant-time
  compare against a plaintext-at-rest bearer key.
- `coord/admin/credentials.go:219` `NewRecoveryCodes` grouped
  `base32.StdEncoding` output and its comment claims an unambiguous alphabet;
  RFC 4648 base32 is not Crockford, so the claim only holds by accident.
- `crypto/rand` failure is handled three incompatible ways: propagate
  (`coord/admin`), return an empty string (`edgeserver/downloadid.go:30`), and
  **fall back to `time.Now().UnixNano()`** (`coord/audit/observer.go:105`).
- `coord/config.go:38` ships `default:"very-strong-password"` for the password
  that Argon2id-derives the key encrypting every custodial wallet private key.

A single audited `secret` package turns a future security fix or CVE response
into a version bump, instead of a repo-by-repo hunt through four idioms.

**Scope, as agreed:** aioz-common only - no depin migration in this change
(depin has several in-flight uncommitted branches); strong extraction
candidates only; full key lifecycle for the new package.

## Part 1 - Extract the clean utility cluster

Twelve packages, ~2,165 LOC, verified to have **zero** `aioz-depin/...`
imports in non-test files and no heavy dependencies (no cosmos-sdk, libp2p,
ethereum, gorm, ghw, gopsutil).

| From (depin) | To (aioz-common) | LOC | Tests | What it is |
| --- | --- | --- | --- | --- |
| `internal/memory` | `memory/` | 298 | yes | `memory.Size` byte-size type, KiB/MiB consts, parse/format, `flag.Value` |
| `internal/date` | `date/` | 70 | yes | month/day boundaries, `PeriodToTime`, `MonthsBetweenDates` |
| `internal/period` | `period/` | 81 | no | `Period` (YYYY-MM) value type with CSV marshal |
| `internal/slices2` | `slices2/` | 45 | yes | generic `Map`, `Convert`, `ConvertErrs` |
| `internal/context2` | `context2/` | 121 | yes | `WithoutCancellation`, `WithRetimeout`, `WithCustomCancel` |
| `internal/signal` | `signal/` | 197 | no | one-shot `Signal` and lazy `Chan` primitives |
| `internal/leak` | `leak/` | 41 | no | resource-leak `Ref` with start-stack capture |
| `internal/lrucache` | `lrucache/` | 257 | yes | `ExpiringLRUOf[T]` with single-flight `Get` |
| `pkg/common/time2` | `time2/` | 498 | no | `Clock`/`Timer`/`Ticker` + fake time machine for deterministic tests |
| `pkg/common/errs2` | `errs2/` | 109 | no | errgroup-like `Group`, `IsCanceled`, `IgnoreCanceled`, `Collect` |
| `pkg/common/base58` | `base58/` | 288 | yes | base58 + base58check (btcd lineage, carries its own LICENSE) |
| `pkg/readcloser` | `readcloser/` | 160 | no | Fatal/Lazy/Limit/Multi `io.ReadCloser` combinators |

### Rules for the move

**Keep the original package names and flatten to top level.** `memory`, `date`,
`time2`, `errs2`, `slices2`, `context2` keep their names even where the `2`
suffix is Storj lineage. This is the same decision already recorded for
`stats/version` and `stats/testcontext`: renaming buys nothing functional and
turns a later depin migration from a pure import-path rewrite
(`aioz-depin/internal/memory` -> `aioz-common/memory`) into an API rewrite.

**Carry history with `git subtree split` per directory**, exactly as `apierr`,
`cycle`, `lifecycle`, `logging` and `stats/monitor` were carried over from
`aioz-template`. Note the known consequence already documented for this repo:
`git blame` resolves correctly, but `git log -- <path>` stops at the import
commit unless you pass `--full-history`.

**Carry `LICENSE` and `README.md` files that travel with a package.**
`pkg/common/base58` has both. depin uses an allowlist-style `.gitignore` that
silently drops new non-`.go` files, so verify the LICENSE actually landed.

**One mechanical refactor is required, not optional:** `errs2` and `readcloser`
use `github.com/zeebo/errs` (v1). `aioz-common`
already depends on `github.com/zeebo/errs/v2`. Convert `errs.Class` to the
v2 `errs.Tag` idiom during extraction rather than adding a second major version
of the same library to the module. Note `errs.Tag` exposes `Errorf` and `Wrap`
- there is no `New`.

**Add a `README.md` per new package**, matching the existing voice (see
`cycle/README.md`, `uuid/README.md`): what problem it solves and which trap it
avoids, then the API. Update the root `README.md` component table and the
Layout tree.

**`zeebo/structs` must stay pinned** at
`v1.0.3-0.20230601144555-f2db46069602`; a `go mod tidy` silently resolves it
down to `v1.0.2`.

### Deliberately not extracted (recorded as follow-up)

- `pkg/common/sync2` and `internal/sync2` are **two live diverged forks** of the
  same package (`cycle.go`, `fence.go`, `nocopy.go`, `sleep.go`, `workgroup.go`,
  `workplace.go` all differ; each side has files the other lacks). They must be
  merged before extraction, and `sync2.Cycle` overlaps `aioz-common/cycle`.
- `pkg/pkcrypto` and `pkg/common/pkcrypto` are likewise **two diverged forks**;
  only the `common` copy has `encoding_test.go`.
- Already duplicated in aioz-common, so these need an API reconciliation rather
  than a move: `pkg/lifecycle` <-> `lifecycle`, `pkg/debug` <-> `stats/debug`,
  `pkg/version` <-> `stats/version`, `internal/testcontext` <->
  `stats/testcontext`, `pkg/testmonkit` <-> `stats/monitor`, `internal/grpcerr`
  <-> `apierr`, `internal/cfgstruct` <-> `config`.
- Needs a decoupling refactor first: `internal/testrand` and `pkg/bloomfilter`
  (both import `internal/vo`), `internal/grpcutil` root (imports `internal/vo`
  in `log.go:8` only), `pkg/utility/hardwareinfo` (pulls `ghw` + `gopsutil`).
- `pkg/checksum` (198 LOC) and `pkg/merkle` (155 LOC) are clean and would
  otherwise qualify, but stay in depin by decision: both are storage-integrity
  primitives serving the piece and segment pipeline, so a shared library is not
  where their next change belongs.
- Too large or too specialised for this module: `pkg/infectious` (11k LOC
  vendored FEC), `pkg/location` (1k LOC of ISO data), `internal/mud` (1.3k LOC
  DI container), `pkg/eestream`.

## Part 2 - The `secret` package

New package `aioz-common/secret`. Stdlib only, no new module dependencies.

### The premise that shapes the API

`go.mod` declares `go 1.25.0`. Since Go 1.24, `crypto/rand.Read` **cannot
return a non-nil error**: if the system entropy source fails, the runtime
terminates the process. All three of depin's incompatible failure policies are
therefore dead branches on its own Go version, including the dangerous one
(`coord/audit/observer.go:105`, fall back to `time.Now().UnixNano()`).

So: **this package returns an error only when it is given something (a prefix,
a string to parse, a column to scan), never when it is asked to make
something.** Generators return no error. A `(T, error)` whose error is provably
always nil grows an untestable branch at every call site; `uuid.Must` in this
repo exists for the same reason. `MinBytes` violations panic, because `n` is a
constant at essentially every call site.

### Two types, and the split is the design

- **`Token`** is plaintext. It redacts itself in `fmt` (every verb),
  `slog`, `String` and `GoString`, and it **errors** on `MarshalText`,
  `MarshalJSON` and `driver.Valuer`. `Reveal()` is the only way out.
- **`Hash`** is what you store. Full `MarshalText`/`MarshalJSON`/`Valuer`/
  `Scanner`, because it is safe to store and safe to log.

A secret flows in freely and out only through `Reveal`.

Two details are load-bearing and easy to get wrong:

- **`fmt.Formatter`, not just `Stringer`.** fmt consults `Stringer` for `v s q
  x X` only; `fmt.Sprintf("%d", tok)` takes the `badVerb` path and reflects out
  the raw string. `Format` is what actually closes fmt.
- **Marshallers must error, not emit a placeholder.** A handler that returns
  the literal `secret.Token(redacted)` ships a bug that looks like a working
  deploy and fails later, elsewhere, untraceably. And `Value()` returning an
  error is the single highest-value method here: `Token`'s kind is `string`, so
  without a `driver.Valuer`, `database/sql`'s reflect fallback **silently
  stores the plaintext** - verbatim the `certificate/authorization` bug.

One hole redaction cannot close: `fmt.Sprintf("%+v", s)` where `s` has an
**unexported** `Token` field. fmt cannot call methods on unexported fields.
This gets a README paragraph and a test that asserts the hole exists.

### API

```go
package secret

var Error = errs.Tag("secret")
const MinBytes = 16                       // 128-bit floor, enforced by panic
func Bytes(n int) []byte

// Token: sessions, CSRF, invites, resets, recovery codes, the presented half of a Key.
const Redacted   = "secret.Token(redacted)"
const TokenBytes = 32
type Token string
func NewToken() (Token, Hash)             // 32 bytes, lowercase hex, 64 chars
func NewTokenN(n int) (Token, Hash)
func (t Token) Reveal() string
func (t Token) Hash() Hash
func (t Token) IsZero() bool
func (t Token) String() string                      // Redacted
func (t Token) GoString() string                    // Redacted
func (t Token) Format(f fmt.State, verb rune)       // Redacted, every verb
func (t Token) LogValue() slog.Value                // Redacted
func (t Token) MarshalText() ([]byte, error)        // always errors
func (t Token) MarshalJSON() ([]byte, error)        // always errors
func (t Token) Value() (driver.Value, error)        // always errors: store the Hash
func (t *Token) UnmarshalText(b []byte) error       // works
func (t *Token) UnmarshalJSON(b []byte) error       // works
func (t *Token) Scan(value any) error               // works

// Hash: lowercase hex SHA-256, unsalted.
type Hash string
func HashOf(presented string) Hash
func Verify(presented string, stored Hash) bool     // the one comparison entry point
func (h Hash) Equal(other Hash) bool                // subtle.ConstantTimeCompare
func (h Hash) Scheme() string                       // "sha256"; else reads a $name$ prefix
func (h Hash) Valid() bool
func (h Hash) IsZero() bool
// + String, MarshalText/JSON, UnmarshalText/JSON, Value, Scan

// Key: public id + secret half, "<prefix>_<id><secret><crc>".
const KeyIDLen, KeySecretLen, KeyChecksumLen = 12, 43, 6   // base62
type Key    struct { Prefix, ID string; Secret Token; Hash Hash }
type KeyRef struct { Prefix, ID string }             // no secret, safe to log
func NewKey(prefix string) (Key, error)              // errors only on a bad prefix
func ParseKey(s string) (KeyRef, error)              // offline shape + checksum

// Recovery codes: Crockford base32, XXXX-XXXX-XXXX-XXXX.
const CodeGroups = 4
func NewCodes(n int) (codes []Token, hashes []Hash)
func NormalizeCode(s string) string                  // upper, strip, O->0, I->1, L->1
func VerifyCode(typed string, stored Hash) bool
```

### The decisions behind it

**Unsalted SHA-256 is deliberate.** bcrypt's slowness defends a low-entropy
secret against offline guessing and a salt defends against a rainbow table; a
256-bit random input has neither attack. Running bcrypt per request would make
session lookup the slowest part of the API for no gain. It also keeps output
**byte-identical to depin's existing `HashToken`**, so `coord/admin` migrates
with no data change.

**Encoding, one per constructor.** Tokens are lowercase hex: `[0-9a-f]{64}` is
a scanner regex with a usable false-positive rate, it is case-insensitive so a
proxy that lowercases cannot corrupt it, it survives URL/cookie/header/JSON/
YAML/shell unescaped, and it matches depin's existing tokens exactly. Key
bodies are base62 (`0-9A-Za-z`), because *fixed prefix + fixed length +
restricted alphabet* is what makes a high-precision gitleaks/trufflehog rule
possible (`aioz_[0-9A-Za-z]{61}`); hex and base64 bodies are unmatchable.
Base62 is generated by **rejection sampling** (draw a byte, discard if `>= 248`,
else `alphabet[b%62]`) - not `math/big`, which loses leading zeros and varies
the width. Rejected base64url everywhere: two alphabets, optional padding,
`-`/`_` break word-selection, case-sensitive through infrastructure that is not.

**Crockford base32 for human-typed codes, and the precise reason.**
`credentials.go:224` claims base32 "has no 0/O or 1/I ambiguity". RFC 4648's
*output* alphabet is `A-Z2-7`, so the claim accidentally holds in one direction
only. It provides **no input folding**: a person reading `O` off paper and
typing `0` is rejected with no recovery path, at the exact moment they have
already lost their phone. Ambiguity is a parsing property, not an alphabet
property. `NormalizeCode` is the actual fix, and `VerifyCode` calls it so a
caller cannot forget.

**The embedded key ID is the structural fix.** `ParseKey` yields the id with no
database, so verification is: parse, `SELECT hash WHERE id = $1`, one
constant-time compare. That is one indexed row, and it is the direct fix for
`certificate/authorization/db.go:114-121`, which loads every authorization in a
group and runs a linear `bytes.Equal` scan. Without an embedded id, a scan over
every stored hash is the only possible implementation. 12 base62 chars is ~71
bits; the id is public, so it needs collision resistance, not unguessability.
A UUID id is rejected: 36 chars and the dashes destroy the one-word property.

**Hash the whole presented string**, prefix included. The verifier never has to
slice correctly, and a key with a mangled prefix fails rather than matching.

**The CRC32 checksum is not a MAC and the doc comment must say so in those
words.** It buys exactly two things: a third-party secret scanner can tell a
real AIOZ key from a random word with no network call, so a leaked key gets
reported without us running a validation endpoint; and our middleware rejects a
truncated key with 400 before a database round trip. A truncated HMAC is
rejected - it needs a server-side key present wherever a key is validated,
which is precisely the parties (GitHub's scanner, a client SDK) that cannot
hold one.

**Rotation costs zero bytes today.** The prefix is mandatory and is inside the
hashed material, so a future scheme mints `aioz2_...` and an old key can never
verify as a new one. On the stored side, `Hash.Scheme()` returns `"sha256"` for
a bare 64-hex value and otherwise reads a `$name$` prefix, so a future `HashOf`
emits `$s3$<hex>` and `Verify` dispatches on the stored value's own shape. No
schema column, no backfill, no dual-write window, about 15 lines now.

**No password hashing.** Go modules are module-scoped, so `golang.org/x/crypto`
would land in every consumer's build graph for a feature one package in one
service uses (a sibling `password/` package does not help - same `go.mod`). It
is also a different problem: password hashing is a *tuning* problem, and
housing both invites someone SHA-256ing a password because `HashOf` was right
there. depin's bcrypt at `coord/admin/credentials.go:90-130` is already
correct, dummy-hash timing path included.

### Scope fence - what must not go in

No password hashing. No encryption of any kind (depin already has
`coord/pkg/aes` and `pkg/encryption`; a second subtly different one in a shared
library gives you two incompatible ciphertexts and no way to tell them apart).
No JWT, signed tokens or macaroons. No TOTP (that would pull `pquerna/otp`;
recovery *codes* are in scope, the TOTP algorithm is not). No secret storage or
sourcing - `config` already owns `secrets.yaml` and its 0600 check. No expiry,
revocation or `Session` type. No `math/rand`, and no non-secret identifiers -
that is `uuid`'s job, and the README states plainly that `uuid.New()` is UUIDv7
(~74 random bits behind a 48-bit millisecond timestamp), a database key and
never a bearer token. No general hashing utility; `HashOf` must never grow a
`Sha256File` sibling.

### Files

```
secret/
├── README.md            # cross-links uuid; names the bcrypt boundary
├── secret.go            # package doc, Error, MinBytes, Bytes
├── token.go             # Token, the redaction method set, NewToken
├── hash.go              # Hash, HashOf, Verify, Equal, Scheme, encoding, sql
├── key.go               # Key, KeyRef, NewKey, ParseKey, prefix validation
├── code.go              # NewCodes, NormalizeCode, VerifyCode, Crockford
├── base62.go            # unexported: rejection sampling, CRC32 codec
└── *_test.go, example_test.go
```

### Implementation order

1. `secret.go` - package doc first; it is the design review.
2. `hash.go`, then `hash_test.go` + `constant_time_test.go`.
3. `token.go` with a `var _ fmt.Formatter = Token("")` compile-time assertion
   block (currency's precedent) covering `Formatter`, `Stringer`, `GoStringer`,
   `slog.LogValuer`, `json.Marshaler`, `encoding.TextMarshaler`,
   `driver.Valuer`, `sql.Scanner`.
4. `token_test.go`. Do not proceed until `%d` and `ConvertValue` both pass.
5. `base62.go` + test, then `key.go` + test, then `code.go` + test.
6. `example_test.go`, `secret/README.md`, root README row + layout tree.

Lint watch-outs: `prealloc` on `NewCodes` slices, `gocritic` on the `Format`
switch, `revive` `filename-format` (snake_case, hence `constant_time_test.go`),
and staticcheck ST1020 - a group-header comment directly above an exported
method is read as its doc comment, so leave a blank line after one.

## Verification

**Extraction (Part 1)**

- `make fmt && make lint && make test` in `aioz-common`, green.
- `go build ./...` and `go vet ./...` at module root.
- `go list -m all | wc -l` before and after: the module is currently 24
  modules; extraction must not raise it (the errs v1 -> v2 conversion is what
  keeps this true). `go mod graph | grep 'zeebo/errs '` must return nothing.
- Confirm every moved package's tests came with it and still pass, and that
  `base58/LICENSE` actually landed (depin's allowlist `.gitignore` silently
  drops new non-`.go` files).
- `git log --full-history -- memory/size.go` shows pre-move history.
- depin is untouched: `git -C /home/tuan/work/depin-workspace/depin status`
  must be unchanged from its current state.

**`secret` package (Part 2)** - the tests are the verification, and four of
them are the point:

1. **Redaction table** (`token_test.go`): a table over every escape channel,
   each asserting the plaintext is absent *and* `Redacted` is present -
   `fmt.Sprintf` with `%v %s %q %x %X %d %#v %+v %8s %-20v`; the value bare, as
   `&tok`, `any(tok)`, in a slice, a map, a struct field, nested two deep;
   `slog.NewJSONHandler` into a buffer; `json.Marshal` -> `require.Error`;
   `driver.DefaultParameterConverter.ConvertValue(tok)` -> `require.Error`
   (stdlib-only, and the actual proof that `database/sql`'s reflect fallback
   cannot reach the plaintext). `%d` is the case that fails without `Formatter`.
   Plus a **positive control** - `Reveal()` returns 64 lowercase hex chars -
   without which the whole file passes trivially against an empty `Token`. Plus
   `TestUnexportedFieldIsNotRedacted`, which asserts the documented hole exists.
2. **Constant-time guard** (`constant_time_test.go`): parse `hash.go` with
   `go/parser` (stdlib), assert `Hash.Equal`'s body calls
   `subtle.ConstantTimeCompare` and contains no `==`/`!=` over the receiver or
   parameter and no `bytes.Equal`/`strings.EqualFold`/`strings.Compare`.
   Deterministic and ~50ms, where a statistical timing test is flaky and proves
   nothing. Paired with behavioural tests so the guard cannot be satisfied by a
   broken implementation.
3. **Golden migration vector** (`hash_test.go`): `HashOf("x")` equals what
   depin's `credentials.go:157 HashToken` produces. This is the test that
   proves the `coord/admin` migration is a no-op.
4. **Exhaustive key mutation** (`key_test.go`): every single-character mutation
   and every truncation offset of a valid key fails `ParseKey`. That is the
   checksum's actual job.

Plus: Crockford folding (`O`/`0`, `I`/`1`, `L`/`1`, lowercase, no dashes,
spaces for dashes) all verify against a minted code; a base62 distribution
check over ~200k characters catching the classic `b % 62` bias; `Bytes(MinBytes-1)`
panics; runnable `Example` functions so the README's redaction claim is
executable documentation.

**Then**: tag `v0.3.0` (currently `v0.2.0`), since this is additive only.

## Follow-ups, explicitly out of this change

- **Highest-severity audit finding, and this package does not fix it**:
  `coord/config.go:38` ships `default:"very-strong-password"` for the password
  that Argon2id-derives the key encrypting every custodial wallet private key
  (`coord/wallet/service.go:58`). The fix belongs in `aioz-common/config`: no
  `default:` on a field carrying a secret, plus a startup refusal to boot on a
  shipped placeholder. File separately so it is not mistaken for handled.
- depin migration: `coord/admin` (byte-identical, no backfill);
  `certificate/authorization` (fixes plaintext-at-rest + non-constant-time
  compare + the O(n) scan at once, but the presented format changes, so
  unclaimed tokens are invalidated - a product decision); recovery codes (**not
  a drop-in**: existing codes were minted from `A-Z2-7` and can contain literal
  `I`, `L`, `O`, which Crockford folds to `1`, `1`, `0`, changing the hashed
  string - re-issue at next MFA enrolment rather than carrying a dual-verify
  path nobody will remember to delete).
- Delete the two dead `crypto/rand`-failure branches in place:
  `edgeserver/downloadid.go:30` and `coord/audit/observer.go:105`.
- Merge the two `sync2` forks and the two `pkcrypto` forks, then reconsider
  extraction.
- `coord/order/signer.go:84` `CreateSerial` fills only `serial[8:]` with random
  bytes (the first 8 are a big-endian expiry timestamp), so a value that also
  serves as a secretbox nonce has 64 bits of randomness, not 128. Separate
  investigation.

---

# Appendix - full depin inventory

The complete scan behind Part 1. Method: every package under `pkg/`,
`internal/`, `coord/pkg/`, `worker/pkg/`; non-test LOC; presence of tests; and
the full set of non-stdlib imports **from non-test files only**. Disqualifier =
a non-test import of `aioz-depin/{coord,worker,edgeserver,internal/vo,pkg/pb}`
or of cosmos-sdk / libp2p / ethereum / ethermint / gorm / ghw / gopsutil.

`go list ./...` was not usable (local `replace` set plus
`coord/admin/ui/node_modules` noise); the import graph was derived by grep,
which is equivalent for this question.

## STRONG - zero domain imports, light deps

Marked **[TAKE]** = selected for Part 1.

| Path | Package | LOC | Tests | Deps | What it is |
| --- | --- | --- | --- | --- | --- |
| `pkg/common/time2` | `time2` | 498 | no | none | **[TAKE]** Clock/Timer/Ticker + fake time machine |
| `pkg/common/sync2` | `sync2` | 1161 | yes | time2, monkit, `calebcase/tmpfile`, errgroup | Cycle, Fence, Event, Limiter, Tee, Copy, WorkGroup, Workplace, Sleep. Overlaps `cycle`; forked |
| `pkg/common/sync2/race2` | `race2` | 85 | no | none (unsafe) | race-detector annotation shims, build-tagged no-ops |
| `pkg/common/errs2` | `errs2` | 109 | no | `zeebo/errs` | **[TAKE]** errgroup-like Group, IsCanceled, IgnoreCanceled, Collect |
| `pkg/common/base58` | `base58` | 288 | yes | none | **[TAKE]** btcd-style base58 + base58check |
| `internal/memory` | `memory` | 298 | yes | none | **[TAKE]** `memory.Size`, KiB/MiB consts, parse/format, flag.Value |
| `internal/date` | `date` | 70 | yes | none | **[TAKE]** month/day boundaries, PeriodToTime, MonthsBetweenDates |
| `internal/period` | `period` | 81 | no | none | **[TAKE]** `Period` (YYYY-MM) value type with CSV marshal |
| `internal/slices2` | `slices2` | 45 | yes | none | **[TAKE]** generic Map, Convert, ConvertErrs |
| `internal/context2` | `context2` | 121 | yes | none | **[TAKE]** WithoutCancellation, WithRetimeout, WithCustomCancel |
| `internal/signal` | `signal` | 197 | no | none | **[TAKE]** one-shot Signal + lazy Chan |
| `internal/leak` | `leak` | 41 | no | none | **[TAKE]** leak `Ref` with start-stack capture, ctx-carried |
| `internal/lrucache` | `lrucache` | 257 | yes | monkit, time2 | **[TAKE]** `ExpiringLRUOf[T]` with single-flight Get |
| `internal/netutil` | `netutil` | 114 | no | monkit, `x/sys/unix` | TCP SetUserTimeout + close-tracking net.Conn wrapper |
| `internal/fpath` | `fpath` | 434 | yes | `zeebo/errs` | atomic file write, temp dirs, editor launch, OS paths |
| `internal/grpcerr` | `grpcerr` | 79 | no | grpc codes/status, errdetails | named gRPC status errors. Conceptual duplicate of `apierr` |
| `internal/grpcutil/peer` | `grpcpeer` | 54 | no | grpc peer/credentials, errs/v2 | typed peer extraction from gRPC context |
| `internal/grpcutil/transport` | `transport` | 70 | no | grpc credentials, zap | passthrough/logging TransportCredentials |
| `internal/grpcutil/cache` | `cache` | 330 | yes | `zeebo/errs` | generic keyed object cache w/ expiry, Take/Put |
| `internal/mud` | `mud` | 1354 | yes (5) | errs/v2, `x/exp/slices`, errgroup, zap | reflection DI container + graphviz dot. Partial overlap w/ `lifecycle` |
| `pkg/readcloser` | `readcloser` | 160 | no | `zeebo/errs` | **[TAKE]** Fatal/Lazy/Limit/Multi io.ReadCloser |
| `pkg/checksum` | `checksum` | 198 | yes (3) | `zeebo/errs` | pluggable 32-bit checksum, streaming + verify |
| `pkg/merkle` | `merkle` | 155 | yes | none | chunked Merkle build + proof/verify |
| `pkg/validator` | `validator` | 65 | no | none | tiny error-map validator (Check, In, Matches, Unique) |
| `pkg/validate` | `validate` | 19 | yes | none | `MetadataValidate` string check; trivial, fold into validator |
| `pkg/utility/osinfo` | `osinfo` | 148 | no | none | OS name/version/arch detection |
| `pkg/pkcrypto` **and** `pkg/common/pkcrypto` | `pkcrypto` | 539 / 560 | yes | `zeebo/errs`, race2 | PEM/DER encode-decode, hashing, ECDSA sign/verify. **Diverged fork pair** - every file differs |
| `pkg/location` | `location` | 1014 | yes | `zeebo/errs` | ISO country codes, continents, regions, CountryCode set |
| `pkg/version/buildinfo` | `buildinfo` | 39 | yes | errs/v2 | ldflags build-info reader. Part of `stats/version` |
| `coord/pkg/aes` | `aes` | 95 | yes | `x/crypto/argon2` | Argon2-derived AES-GCM encrypt/decrypt |
| `coord/pkg/formatter` | `formatter` | 86 | yes | none | ANSI colorize + human duration formatting |
| `coord/pkg/router` | `router` | 82 | no | `julienschmidt/httprouter` | RouteGroup/middleware sugar |
| `worker/pkg/du` | `du` | 134 | no | none (syscall) | cross-platform disk usage |
| `worker/pkg/cron` | `cron` | 58 | no | `robfig/cron/v3` | thin Runner/System cron wrapper. Overlaps `cycle` |
| `pkg/infectious` | (none) | 11270 | no | `x/sys/cpu` | vendored Reed-Solomon / Berlekamp-Welch FEC, mostly generated tables + asm |

### Why the 24 unselected STRONG rows were left out

- **Forked, must be merged first**: `pkg/common/sync2` + `internal/sync2`;
  `pkg/pkcrypto` + `pkg/common/pkcrypto`.
- **Duplicates something aioz-common already has**, so it needs an API
  reconciliation rather than a move: `grpcerr` (vs `apierr`), `buildinfo` (vs
  `stats/version`), `worker/pkg/cron` and `sync2.Cycle` (vs `cycle`), `mud`
  (vs `lifecycle`).
- **Would add a dependency the module does not have**: `coord/pkg/aes`
  (`x/crypto`), `coord/pkg/router` (`httprouter`), `worker/pkg/cron`
  (`robfig/cron`), `netutil` (`x/sys/unix`), the whole `grpcutil` family
  (grpc + zap). Worth taking later, but each is a deliberate dependency
  decision, not a lift.
- **Too large or too specialised for a general library**: `infectious` (11k
  LOC FEC - its own module if anything), `location` (1k LOC of ISO data),
  `mud` (1.3k LOC DI container).
- **Clean, but storage-integrity primitives that belong beside the piece
  pipeline**: `checksum`, `merkle`.
- **Judgement calls, low value**: `race2` (exists only to serve sync2 and
  pkcrypto, neither of which we are taking), `fpath` (the editor-launch half
  does not belong in a library), `validate` (19 LOC, fold into `validator`),
  `validator`, `osinfo`, `du`, `formatter`.

## WEAK - generic in spirit, needs a refactor or a dedup first

| Path | Package | LOC | Tests | Blocker |
| --- | --- | --- | --- | --- |
| `internal/sync2` | `sync2` | 733 | yes | Fork of `pkg/common/sync2`: `cycle.go`, `fence.go`, `nocopy.go`, `sleep.go`, `workgroup.go`, `workplace.go` all differ; `internal` adds `cooldown.go`, `pkg/common` adds copy/event/limiter/tee |
| `pkg/lifecycle` | `lifecycle` | 209 | yes | Clean deps, but duplicates aioz-common `lifecycle` |
| `pkg/debug` | `debug` | 682 | yes | `panel.go:14` imports `internal/sync2`; `server.go:12` imports `pkg/version`. Duplicates `stats/debug` |
| `pkg/version` | `version` | 550 | yes | Encodes AIOZ release/rollout semantics; duplicates `stats/version`; `stats.go:17` registers `version_info` |
| `internal/testcontext` | `testcontext` | 336 | no | `context.go:15` imports `internal/memory`. Duplicates `stats/testcontext` |
| `internal/cfgstruct` | `cfgstruct` | 730 | yes | cobra+pflag reflection config binder; overlaps aioz-common `config` |
| `internal/grpcutil` (root) | `grpcutil` | 415 | yes | `log.go:8` imports `internal/vo`; conntrack/stop halves are generic - needs a split |
| `internal/grpcutil/pool` | `pool` | ~1000 | yes (3) | imports `grpcutil/cache` + `internal/peertls/tlsopts` |
| `internal/grpcutil/grpcconn` | `grpcconn` | 323 | yes | imports `grpcutil/connector` (libp2p) + `pool` |
| `pkg/testmonkit` | `testmonkit` | 136 | yes | overlaps `stats/monitor` |
| `internal/testrand` | `testrand` | 244 | yes | imports `internal/vo` + `internal/memory`; generic half (Intn/Bytes/Reader) is extractable |
| `pkg/bloomfilter` | `bloomfilter` | 252 | yes | `filter.go:10-11` import `internal/memory` **and** `internal/vo` (keyed on `vo.PieceID`) |
| `pkg/ers_schema` | `ers_schema` | 278 | yes | imports `pkg/infectious` + race2 |
| `pkg/eestream` | `eestream2` | 1416 | yes | imports fpath, memory, ers_schema, infectious; `stripe.go:310` hardcodes a `storj.io/storj/uplink/eestream` monkit scope |
| `pkg/utility/hardwareinfo` | `hardwareinfo` | 658 | no | `jaypipes/ghw` + `shirou/gopsutil/{cpu,host,mem}` |
| `pkg/common` (root) | `common` | 297 | yes | imports `pkg/pb/shared` |
| `pkg/encryption` | `encryption` | 797 | yes | imports `pkg/common` (pb-bound) + readcloser |
| `coord/pkg/ratelimiter` | `ratelimiter` | 68 | no | imports `internal/lrucache` + `x/time/rate`; extractable once lrucache lands |
| `worker/pkg/httpserver` | `httpserver` | 207 | yes | imports `internal/grpcutil` + `pkg/lifecycle` |
| `worker/pkg/{space,sysinfo,filemanager,speedestimation}` | - | 59-143 | no | each imports a depin package; thin domain wrappers |
| `pkg/deprecated_encrypt` | `deprecated_encrypt` | 97 | yes | clean deps but explicitly deprecated - do not carry into a shared lib |

## NO - domain-bound

| Path | Why |
| --- | --- |
| `internal/vo` | 5426 LOC, the domain value-object package: cosmos-sdk codec/types, ethermint ethsecp256k1, go-ethereum common, libp2p peer |
| `internal/billing`, `internal/compensation`, `internal/fx` | import `internal/vo`, `internal/period`, `cosmossdk.io/math` - payout/accounting domain |
| `internal/testplanet` | 14029 LOC; imports ~50 `coord/...`, `worker/...`, `edgeserver` packages |
| `internal/version/checker` | imports `versioncontrol/{api,config,usecase}`, `worker/pkg/{keytool,p2pc}`, `internal/vo` |
| `internal/peertls`, `.../extensions`, `.../tlsopts`, `internal/revocation` | node-identity/peer-cert model; import `pkg/identity`, `pkg/pkcrypto`, `internal/vo` |
| `internal/p2putil`, `internal/p2pmonitor`, `internal/grpcutil/connector{,/p2p}`, `internal/grpcutil/dial` | libp2p host/swarm/rcmgr + `pkg/identity` + `internal/vo` |
| `pkg/auth`, `pkg/helper`, `pkg/encoding` | cosmos-sdk + ethermint + go-ethereum crypto; `pkg/encoding` also imports `AIOZNetwork/go-aioz/app` |
| `pkg/identity`, `pkg/identity/testidentity`, `pkg/nodetag`, `pkg/signing` | node identity / signed-tag domain |
| `pkg/p2p`, `pkg/protocol` | libp2p host/transport; worker registration DTOs |
| `pkg/pb/**` | generated protobufs |
| `pkg/process` | 2173 LOC; cosmos-sdk, ethermint, libp2p gologshim, viper/cobra/cast, logship, version |
| `pkg/logship`, `pkg/telemetry` | import `pkg/lifecycle` + `pkg/pb/telemetry` |
| `pkg/modular/config` | imports `internal/mud`, `internal/cfgstruct`, `pkg/process` |
| `coord/pkg/responses` | imports `coord/pkg/formatter` + `internal/vo` |
| `worker/pkg/{keytool,p2pc,signaturecheck,coordstore,hashstore}` | cosmos/ethermint keyring, libp2p, `pkg/signing`, `internal/vo` |

## Cross-cutting findings

1. **Two live forks of `sync2`**: `internal/sync2` (733 LOC) and
   `pkg/common/sync2` (1161 LOC), every shared file differs.
   `pkg/debug/panel.go:14` uses the `internal` one; `pkg/logship` and
   `pkg/telemetry` use the `pkg/common` one.
2. **Two live forks of `pkcrypto`**: `pkg/pkcrypto` and
   `pkg/common/pkcrypto`, all five files differ; only the `common` copy has
   `encoding_test.go`.
3. **Eight packages already duplicate aioz-common**: `pkg/lifecycle` <->
   `lifecycle`, `pkg/debug` <-> `stats/debug`, `pkg/version` +
   `pkg/version/buildinfo` <-> `stats/version`, `internal/testcontext` <->
   `stats/testcontext`, `pkg/testmonkit` <-> `stats/monitor`, `sync2.Cycle`
   <-> `cycle`, `internal/grpcerr` <-> `apierr`, `internal/cfgstruct` <->
   `config`.
4. **No `uuid` or `currency` equivalent exists in depin** - `internal/vo` owns
   UUID and is domain-bound.

---

# Part 3 - The config secret guard (done, was listed as a follow-up)

The follow-up flagged as the audit's highest-severity item was implemented in
the same branch, because it lives in `aioz-common/config` and the `secret`
package would otherwise look like it had addressed it.

`depin/coord/config.go:38` ships `default:"very-strong-password"` for the
password that Argon2id-derives the key encrypting every custodial wallet
private key (`coord/wallet/service.go:58`). Being a `default`, it reaches
release builds.

**`config/secrets.go`**, two halves that only work together:

- `secret:"true"` / `secret:"optional"` on a field. A `default` or
  `releaseDefault` on it panics at bind time, naming the field, the flag and
  the offending value. Matches the existing `internal:"true"` idiom.
- `CheckSecrets(&cfg)` after `Load` reports every required secret still unset,
  with field path, flag and help text, so the service refuses to boot.

Without the first, `CheckSecrets` passes on the placeholder. Without the
second, deleting the placeholder silently yields an empty key.

Gotcha that shaped the API: `getDefault` already panics when `devDefault` has
no `releaseDefault`, so a secret with a local convenience value is written
`secret:"true" devDefault:"dev-only" releaseDefault:""`. The empty-but-present
tag satisfies both rules and states in the tag that nothing ships.

Verified: `config` coverage 84.4%, whole module lint-clean, `go vet` clean,
suite green with `-race -count=1`, depin still untouched. `TestBindRejectsAShippedSecret`
reproduces depin's exact struct shape.
