# configuration — struct tags, cfgstruct, mud DI, modular (start here)

How a DePIN peer turns a plain Go config struct into command-line flags, a YAML file,
and finally a wired-up graph of running components. This is the onboarding guide:
the flow and what to expect. For the components this wiring produces (the transport)
see [`transport.md`](./transport.md).

> Three independent mechanisms stack here: **`cfgstruct`** reflects struct tags into
> flags, **`mud`** is a dependency-injection container, and **`pkg/modular/config`**
> is the glue that registers config structs *into* mud so the container can bind and
> inject them. You can understand each in isolation.

---

## 1. What it is & why

- **`cfgstruct`** (`internal/cfgstruct`) — reflects over a config struct and, for each
  field, creates a `pflag` flag from struct tags (`help`, `default`, `devDefault`,
  `testDefault`, `user`, `path`, …). It also expands `$CONFDIR` / `$IDENTITYDIR` /
  `$DATADIR` tokens inside default values.
- **`mud`** (`internal/mud`) — a small DI container ("**M**odular **U**niversal
  **D**ependency"). You `Provide` constructors keyed by their return type; mud resolves
  the dependency graph and runs each component's `Run`/`Close` lifecycle. See
  [`../internal/mud/README.md`](../internal/mud/README.md).
- **`pkg/modular/config`** — bridges the two: `RegisterConfig[T]` puts a `*T` config
  into the mud ball (tagged with its flag prefix), and `BindAll` walks every such
  component and calls `cfgstruct.Bind` on it.

Why three layers? Tags-to-flags, dependency-injection, and "bind every registered
config" are genuinely separate concerns. Keeping them apart means a subsystem declares
its config struct once and gets flags + file loading + injection for free.

---

## 2. Mental model

```
   Config struct  (fields with `help:"" default:"" devDefault:"" user:"" path:"" ...`)
        │
        │  cfgstruct.Bind(flags, &cfg, ConfDir(...), IdentityDir(...), Prefix("worker"))
        ▼
   Cobra / pflag FlagSet   ── $CONFDIR/$IDENTITYDIR/$DATADIR expanded in defaults
        │
        │  viper loads config.yaml, merges file > flag-default into the struct
        ▼
   populated Config struct
        │
        │  modular/config.RegisterConfig[T](ball, "worker")   — config enters the ball
        ▼
   mud Ball (registry of Components)
        │  Provide[T](ball, factory)        — constructors keyed by return type
        │  mud resolves deps by type, injecting Config + peers
        ▼
   component init  →  Run(ctx)  ...  Close(ctx)     (lifecycle, reverse-order close)
```

cfgstruct fills the struct; mud decides *who needs it* and constructs the graph.

---

## 3. Glossary

| Term | Meaning | Where |
| ---- | ------- | ----- |
| **`Bind`** | reflect a config struct into pflag flags from struct tags | `internal/cfgstruct/cfgstruct.go` |
| **`BindOpt`** | option to `Bind`: `ConfDir`, `IdentityDir`, `DataDir`, `Prefix`, `UseTestDefaults`, … | `cfgstruct.go` |
| **struct tag** | `help` / `default` / `devDefault` / `testDefault` / `user` / `path` on a field | `cfgstruct.go` |
| **`$CONFDIR` etc.** | tokens expanded inside a default value to the resolved dir | `cfgstruct.go` |
| **`Ball`** | the mud registry: a list of `*Component` | `internal/mud/mud.go` |
| **`Component`** | one registered constructor + its deps + lifecycle stages | `internal/mud/component.go` |
| **`Provide`** | register a constructor keyed by its return type | `mud.go` `Provide` |
| **`View`** | register a cheap adapter from one provided type to another | `mud.go` `View` |
| **`Tag`** | attach metadata to a component (used to mark config components) | `mud.go` `Tag` |
| **`ForEachDependency`** | walk a target's transitive deps, invoking a callback | `mud.go` `ForEachDependency` |
| **`RegisterConfig`** | put a `*T` config into the ball, tagged with its prefix | `pkg/modular/config/config.go` |
| **`BindAll`** | for every config-tagged component, call `cfgstruct.Bind` | `config.go` `BindAll` |
| **dev/test default** | mode-specific default chosen by `UseDevDefaults`/`UseTestDefaults` | `cfgstruct.go` |

---

## 4. Step-by-step walkthrough

### Step 1 — declare the config with struct tags

A subsystem declares one struct; each field's tags drive the flag. From
`pkg/identity/certificate_authority.go`, a real example:

```go
Difficulty uint64 `default:"36" help:"minimum difficulty for identity generation"`
```

`cfgstruct.Bind` understands `help`, `default`, `devDefault`, `testDefault`,
`releaseDefault`, `user` (writable to the config file), `path` (rewritten for the OS),
plus `hidden`, `deprecated`, `noflag`, `flagname`, and `noprefix`.

### Step 2 — bind struct → flags (`cfgstruct.Bind`)

`Bind(flags, &cfg, opts...)` reflects over `cfg` and registers a flag per field.
`BindOpt`s control directories and naming:

- `ConfDir(dir)`, `IdentityDir(dir)`, `DataDir(dir)` set the `$CONFDIR`,
  `$IDENTITYDIR`, `$DATADIR` tokens that get **expanded inside default values** (so a
  default like `$CONFDIR/config.yaml` resolves to the real path).
- `Prefix("worker")` prefixes every generated flag name.
- `UseDevDefaults()` / `UseTestDefaults()` / `UseReleaseDefaults()` pick which of the
  mode-specific defaults wins.

The peer entrypoints call this (via the `process.Bind` wrapper) with
`Prefix(...)`, `ConfDir(defaultConfigDir)`, `IdentityDir(defaultIdentityDir)`. After
`Bind`, viper loads `config.yaml` from the config dir and merges file values over the
flag defaults into the struct — so the struct is fully populated before any component
sees it.

### Step 3 — register the config into mud (`RegisterConfig`)

`RegisterConfig[T](ball, "worker")` (`pkg/modular/config/config.go`) provides a `*T`
(and a value `T` via `View`) into the ball and `Tag`s the component with a
`Config{Prefix: "worker"}` marker so it can be found later.

### Step 4 — bind every registered config (`BindAll`)

`BindAll(ctx, cmd, ball, selector, opts...)` iterates the components tagged `Config`,
and for each calls `Bind(...)`, which inits the component (reads the `*T` pointer out
of mud), appends that component's prefix to the `BindOpt`s, and delegates to
`cfgstruct.Bind`. This is how *every* subsystem's config struct gets its flags without
the main function naming them one by one.

### Step 5 — provide the components and resolve the graph (`mud`)

Subsystems register their constructors with `Provide[T](ball, factory)`. mud keys each
component by its **return type** and resolves arguments by type — so a constructor that
takes `worker.Config` automatically receives the populated config registered in step 3,
and a constructor that takes a `*Dialer` receives whatever provided one. `View` adds a
cheap type adapter; `Tag` attaches metadata. `ForEachDependency(ball, target, cb, …)`
walks the transitive dependency set in order.

### Step 6 — lifecycle

mud detects `Run(ctx) error` and `Close(ctx) error` methods on components and drives
them: it inits the selected component and its dependencies, runs them (background runs
go into an errgroup), and on shutdown closes them in **reverse** order. This mirrors the
peer lifecycle described in [`../SOURCE_STRUCTURE.md`](../SOURCE_STRUCTURE.md): the
Peer is the composition root, subsystems below it.

---

## 5. Key types / entry points

```go
// cfgstruct (internal/cfgstruct/cfgstruct.go)
func Bind(flags FlagSet, config any, opts ...BindOpt)
func ConfDir(path string) BindOpt
func IdentityDir(path string) BindOpt
func DataDir(path string) BindOpt
func Prefix(prefix string) BindOpt
func UseDevDefaults() BindOpt
func UseTestDefaults() BindOpt

// mud (internal/mud/mud.go)
func NewBall() *Ball
func Provide[A any](ball *Ball, factory interface{}, options ...any)
func View[From any, To any](ball *Ball, convert func(From) To)
func Tag[A any, Tag any](ball *Ball, tag Tag)
func ForEachDependency(ball *Ball, target ComponentSelector,
    cb func(*Component) error, selectors ...ComponentSelector) error

// modular glue (pkg/modular/config/config.go)
func RegisterConfig[T any](ball *mud.Ball, prefix string)
func BindAll(ctx, cmd *cobra.Command, ball *mud.Ball,
    selector mud.ComponentSelector, opts ...cfgstruct.BindOpt) error
```

To see the whole flow live, read a `cmd/*/main.go` (e.g. `cmd/worker/main.go`): it
sets up the config/identity dirs, calls `process.Bind(cmd, cfg, defaults,
Prefix(...), ConfDir(...), IdentityDir(...))`, and lets the Cobra `RunE` populate the
structs before constructing the peer.

---

## 6. Gotchas / edge cases

- **Three independent systems, one pipeline.** cfgstruct does *not* know about mud and
  vice versa — `pkg/modular/config` is the only thing that joins them. Don't look for
  flag logic inside mud.
- **`$CONFDIR`/`$IDENTITYDIR`/`$DATADIR` only expand inside default values**, and only
  if you passed the matching `ConfDir`/`IdentityDir`/`DataDir` `BindOpt`. A default
  referencing a token you didn't set will not resolve.
- **Mode-specific defaults are mutually exclusive at bind time.** Choosing
  `UseDevDefaults` vs `UseTestDefaults` vs release decides which of `devDefault` /
  `testDefault` / `releaseDefault` / `default` is applied; cfgstruct will panic if a
  field declares an opposite-mode default but no usable one for the active mode.
- **`user:"true"` controls file persistence**, not visibility — it marks a field as one
  that may be written back to the config file.
- **mud resolves by type, not by name.** Two components that return the same Go type
  collide; use a distinct named type or a `View` adapter to disambiguate.
- **Close runs in reverse init order.** A component that grabs a resource in its
  constructor must release it in `Close`, since mud will not unwind constructor side
  effects for you.

---

## 7. Where to go next

- [`../internal/mud`](../internal/mud) and [`../internal/mud/README.md`](../internal/mud/README.md)
  — the DI container in depth (`Ball`, `Component`, `Provide`, lifecycle).
- [`../pkg/modular`](../pkg/modular) — the `RegisterConfig` / `BindAll` glue.
- [`transport.md`](./transport.md) — a concrete graph of components this wiring builds.
- [`identity-trust.md`](./identity-trust.md) — how identity/TLS options are configured
  and injected.
- [`../SOURCE_STRUCTURE.md`](../SOURCE_STRUCTURE.md) — the Peer → Subsystem layering the
  container assembles.
