---
type: fact
tags: [go, aioz-config, cobra, gotcha]
created: 2026-09-04
agent: main
---

`aioz-config`'s `BindFlags` treats an **anonymous (embedded) struct field as flattened with the parent's prefix**, not with its own (`bind.go:294-306` recurses on `field.Anonymous` using the same prefix). A **named** field contributes its name as the prefix.

So embedding `debug.Config` from aioz-stats produces a bare `--addr`, which collides with a service's own `--server.address` intent; `Debug debug.Config` produces `--debug.addr` / `AIOZ_DEBUG_ADDR` / `debug.addr`. `aioz-template` uses the named form for exactly this reason, and its `internal/config` test asserts `flags.Lookup("addr") == nil`.

Separately: an embedded struct still has to be an **exported type**, because reflection cannot `Interface()` an unexported embedded field.

**How to apply:** embed only to deliberately merge a namespace; use a named field to nest one. Bind the whole config struct in a unit test - an unsupported field type panics at bind time, and that test is the only thing that turns the panic into a `make test` failure rather than a production startup crash.

Applies to [[config-package-extraction]] and [[service-template-scaffold]].
