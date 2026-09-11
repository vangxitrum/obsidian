---
type: fact
tags: [go, build, ldflags, gotcha]
created: 2026-09-04
agent: main
---

`go build -ldflags "-X pkg.SomeStruct.Field=value"` is a **silent no-op**. The linker's `addstrdata1` splits at the last dot and looks up a symbol literally named `pkg.SomeStruct.Field`, which does not exist; `addstrdata` then does `if s == 0 { return }` with no diagnostic. `-X` only patches a symbol whose type is exactly `string`.

Verified empirically 2026-09-04 - a test binary with both forms printed `struct="" plain="PLAINVAL"`.

**Consequence:** `aioz-stats`' README documents exactly this broken form (`-X .../version.Build.Release=...`), so any binary following it reports empty version fields with no build error anywhere.

**How to apply:** keep ldflags targets as plain package-level `var x = "dev"` strings, and copy them into whatever struct needs them at runtime. `aioz-template` does this in `internal/version` (`release`/`build`/`sha` + `Register()`), and its e2e suite asserts the stamp reaches both `version --json` and the debug server's `/version/`, so the trap cannot regress unnoticed.

Applies to [[service-template-scaffold]].
