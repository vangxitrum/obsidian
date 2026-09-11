---
type: decision
tags: [aioz-common, config, security, go]
created: 2026-09-08
agent: main
---

2026-09-08, same branch as [[aioz-common-depin-extraction]]: `aioz-common/config`
gained a `secret` struct tag and a `CheckSecrets` startup check.

The trigger: `depin/coord/config.go:38` ships
`default:"very-strong-password"` for the password that Argon2id-derives the key
encrypting every custodial wallet private key (`coord/wallet/service.go:58`).
Because it is a `default:` it reaches release builds, so it is the password of
every deployment that never overrode it, and it sits in every clone, image and
CI log. It is not a weak secret, it is a public one. The
[[aioz-common-secret-package]] cannot fix this - the problem is in the
configuration layer.

Two halves, and NEITHER WORKS ALONE. Without the bind-time half,
`CheckSecrets` passes on the placeholder; without `CheckSecrets`, deleting the
placeholder silently yields an empty key.

1. `secret:"true"` / `secret:"optional"` on a field: a `default` or
   `releaseDefault` on it panics at bind time, naming field, flag and the
   offending value. Follows the existing `internal:"true"` idiom
   (`config/bind.go:267`), and the naming requirement comes from
   [[cfgstruct-bindable-types]]-style experience: a panic in a command's
   `init()` takes down every role of the binary, so the message is all the
   operator gets.
2. `config.CheckSecrets(&cfg)` after `Load` reports EVERY unset required
   secret at once (field path + flag + help text), so a service refuses to
   boot. All at once because one at a time turns one deployment fix into as
   many restarts as there are missing values.

GOTCHA that shaped the API: `getDefault` already panics if you set
`devDefault` without a `releaseDefault` ("opposites" check,
`config/bind.go:487`). So a secret with a dev convenience value MUST be written
as:

    Password string `secret:"true" devDefault:"dev-only" releaseDefault:""`

An empty-but-present `releaseDefault` satisfies both rules (`tag.Lookup`
returns ok for an empty value) and makes "nothing ships" explicit in the tag.

`findMissingSecrets` deliberately re-implements bind's flagname derivation
(`hyphenate(snakeCase(name))`, prefix per named struct, embedded structs
flattened) rather than reusing `bindConfig`, which is entangled with pflag.
The duplication is intentional so `CheckSecrets` works on a plain struct with
no cobra/pflag state; a test pins the embedded-struct case to bind's behavior.

Lint note: golangci's govet `inline` check rejects `reflect.Ptr`; use
`reflect.Pointer`.
