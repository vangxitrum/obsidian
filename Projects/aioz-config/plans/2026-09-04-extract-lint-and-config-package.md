# Extract CI/CD lint + depin config loader into aioz-config package

## Context

`/home/tuan/work/aioz-config` is an empty repo (README only, `main`, 1 commit, remote
`ssh://git@gitlab.internal/tuan.quang.tran/aioz-config`). Two pieces of working infrastructure live where
other repos cannot import them:

1. **CI/CD lint** exists only as copy-paste templates in `/home/tuan/work/team-playbook/templates/golangci-lint/`.
   `/home/tuan/work/templates/aioz-stats` has already adopted them and fixed the template's rough edges
   (the dangling `<<: *on_code_change` anchor replaced with real `rules:`, the hardcoded
   `!**/internal/utils/log/**` depguard exclusion dropped, a `test` Makefile target added). **aioz-stats is
   the copy source, not the playbook.**
2. **Config loading** lives inside `/home/tuan/work/depin-workspace/depin` as coupled internal packages
   (`internal/cfgstruct`, the config half of `pkg/process`, plus `internal/fpath`). Storj-derived, and it
   already does what is wanted: struct-tag defaults → pflag → viper → YAML file + env, and generates a
   default `config.yaml` with unset values commented out. Module is `aioz-depin` and the packages are
   `internal/`, so nothing can import it.

Outcome: a standalone module `gitlab.internal/tuan.quang.tran/aioz-config` that any AIOZ Go service can
`go get` for config loading and default-config generation, linted by the team lint setup applied to itself.

**User decisions:** faithful port of depin's system (cobra + viper + pflag hard deps, not a rewrite);
lint applied to this repo only, no `templates/` surface; flat root package; clean up the merged API surface
even where that costs depin drop-in compatibility; fix the two decode-loop bugs rather than reproduce them.

## Part 1 — the config package

Flat root, `package config`, module `gitlab.internal/tuan.quang.tran/aioz-config`, **`go 1.25.0`** (matches
aioz-stats and the `registry:5000/go:1.25` CI image; no separate `toolchain` line). Nothing ported needs
past Go 1.21, but 1.25.0 keeps the fleet consistent.

### Files

| File | Ported from | Contents |
| --- | --- | --- |
| `bind.go` | `internal/cfgstruct/cfgstruct.go` | reflection walk, struct-tag defaults, `$CONFDIR`/`$IDENTITYDIR`/`$DATADIR` expansion, annotations, `BindFlags`, `BindOption` constructors |
| `defaults.go` | same | `DefaultsType`, `DefaultsFlag`, `SetupFlag`, `FindFlagIn`/`FindFlagEarly`, `SetRelease` |
| `case.go` | `internal/cfgstruct/case.go` | `snakeCase`, `hyphenate` |
| `flagset.go` | `internal/cfgstruct/flag.go` | `FlagSet` interface |
| `registry.go` | `pkg/process/exec_conf.go` globals | `Registry`, `New`, package-default delegation, `Bind`, `Forget` |
| `load.go` | `pkg/process/exec_conf.go` config half + `exec.go` | `Viper`, `ViperWithCustomConfig`, `ReadConfigFiles`, `Load`, `LoadResult`, `LoadOption`, `fileExists` |
| `save.go` | `pkg/process/config.go` | `SaveConfig`, `SaveOption` |
| `paths.go` | `internal/fpath/atomic.go`, `os.go` | `AtomicWriteFile`, `ApplicationDir`, `IsValidSetupDir`, `IsWritable` |
| `README.md` | `internal/cfgstruct/README.md` (21 KB tag reference) + new usage/migration sections | |

Left behind entirely: `Exec`/`ExecWithCustomOptions`/`InitBeforeExecute`, `Ctx`, `AtomicLevel`, logger /
logship / tracing / debug-server / signal wiring, monkit, and `exec.go`'s `init()` that mutates
`cobra.MousetrapHelpText` (a CLI-main concern, not a library's).

### API after the cleanup

Only real Go-level collision between the two source packages is `Bind`; everything else is disjoint.
Renames chosen for clarity:

- `Bind(cmd *cobra.Command, config any, opts ...BindOption)` — cobra-level, keeps the name (~30 depin call sites).
- `BindFlags(flags FlagSet, config any, opts ...BindOption)` — was `cfgstruct.Bind`, the no-cobra path
  (depin's testplanet uses it: `internal/testplanet/coord.go:83`, `worker.go:71`, `relay.go:86`).
- `ReadConfigFiles(cmd, vip) error` — was `process.LoadConfig`. Reads `config.yaml` then merges
  `secrets.yaml`. Renamed because the new `Load` does something entirely different; `Load`/`LoadConfig`
  side by side is a trap. Its function type is exported as `ConfigLoader` for `ViperWithCustomConfig`.
- `FlagPrefix(string) BindOption` — was `cfgstruct.Prefix`; frees the name for the viper env prefix.
- Option types unified as `BindOption` / `LoadOption` / `SaveOption`, with `type BindOpt = BindOption` and
  `type SaveConfigOption = SaveOption` aliases so existing call sites still compile.
- `Registry` replaces the package-level `configs`/`vipers`/`commandMtx` maps; package-level `Bind`/`Load`/
  `Viper`/`SaveConfig` delegate to a default `Registry`, so simple consumers see no change.

New decode entry point, extracted from `InitBeforeExecute`'s wrapped `RunE`:

```go
type LoadResult struct {
    Viper       *viper.Viper
    UsedKeys    []string
    MissingKeys []string // sorted
    BrokenKeys  []string // sorted
}
func (r *Registry) Load(cmd *cobra.Command, opts ...LoadOption) (*LoadResult, error)
```

Sequence, verbatim from depin: `ViperWithCustomConfig` → for each bound config `structs.Decode(vip.AllSettings(), config)`
→ collect used/missing/broken → propagate missing keys onto pflags/stdlib flags → subtract used from missing.
`LoadOption`s: `WithConfigLoader(ConfigLoader)`, `WithFailOnValueError()`, `WithLogger(*slog.Logger)`.
Returning the key sets (rather than only logging them) is what lets a consumer decide; depin's
`type: "helper"` annotation gate on the missing-key log is preserved. Sort the sets — the original ranges
over maps, so its log order is nondeterministic.

`Load` must be called from `RunE`/`PreRunE`/`PersistentPreRunE`: `ReadConfigFiles` reads
`cmd.Flags().Lookup("config-dir").Value.String()` eagerly, and cobra parses flags before those hooks. Guard
with an explicit error if `!cmd.Flags().Parsed()` rather than silently loading nothing.

Consumer wiring:

```go
config.Bind(runCmd, &cfg, config.DefaultsFlag(rootCmd), config.ConfDir(defaultConfigDir))
// in runCmd.RunE:
res, err := config.Load(cmd)
// generation, in the setup subcommand:
err = config.SaveConfig(cmd, filepath.Join(configDir, "config.yaml"))
```

### Must-fix hazards (a verbatim port is not safe as a library)

1. **`fileExists` (`pkg/process/exec.go:25`) calls `log.Fatalf`** — `os.Exit(1)` in the host process because
   a stat returned `EACCES`, and `log` is a denied import under the repo's own depguard rule. Becomes
   `fileExists(path string) (bool, error)`; `ReadConfigFiles` returns the error.
2. **`traceOut` (`exec_conf.go:227`) is a package-level `flag.String("debug.trace-out", ...)`** — runs at
   *import* time and registers on `flag.CommandLine`. Two packages importing this module → "flag redefined"
   panic. Its only reader was monkit. Dropped.
3. **Do not port `pflag.CommandLine.AddGoFlagSet(flag.CommandLine)` (`exec_conf.go:113`)** — under `go test`
   that pulls `test.v`/`test.timeout`/… into `cmd.Flags()` → viper → and `SaveConfig` writes
   `test.timeout: 0s` into the generated config. Caller's `main()` can do it if it wants.
4. **`ViperWithCustomConfig` holds `commandMtx` across the user-supplied `loadConfig` callback**
   (`exec_conf.go:175-201`). `sync.Mutex` is not reentrant, so a custom loader that calls `Viper`/`Bind`
   deadlocks. Lock only around map read and map write; build the viper unlocked, resolve the double-build
   race with check-again-under-lock or a per-command `sync.Once`.
5. **`configs`/`vipers` are never deleted** (unlike `contexts`/`cancels`, cleaned at `exec_conf.go:366-371`),
   pinning each command, its flag set, and the whole config struct graph forever. `Registry` + `Forget`
   fixes the leak and gives tests isolation — today a second `Viper(cmd)` silently returns the memoized
   viper and **drops the options passed to it**.
6. **Decode-loop bug A:** `f.Changed = val != f.DefValue` is assigned even when `f.Value.Set(val)` failed, so
   `SaveConfig` then persists the unparseable value *live* into the regenerated config. Move it into the
   success branch.
7. **Decode-loop bug B:** missing keys are read with `vip.GetString(key)` regardless of type. For a
   `[]string`, viper's cast errors and returns `""`, so the flag is `Set("")` and the list is silently
   emptied. Type-switch on `f.Value.Type()` and use `vip.GetStringSlice` / `pflag.SliceValue.Replace`.

Both decode bugs are behaviour changes vs depin and get called out in the README plus a regression test each.

### Other deliberate deviations

- **Drop `go.uber.org/zap`** (used only so `SetupFlag` can log one error) → `log/slog`. Mandatory: the
  repo's own `.golangci.yml` depguard denies zap. `SetupFlag` loses its `*zap.Logger` parameter.
- **Drop `monkit`**, and **drop `zeebo/errs/v2`** → `fmt.Errorf(... %w)` / `errors.Join`.
- **Single YAML lib:** `SaveConfig` moves from `yaml.v2` to `yaml.v3`. Two real output differences, both
  verified in the module cache: v3 hard-codes 4-space indent (`encode.go:60-63`) vs v2's 2
  (`emitterc.go:284`), which shifts every `[]string` value in every generated config; and v3's resolver no
  longer treats `yes`/`no`/`on`/`off` as booleans, so a *string* with one of those values is emitted
  unquoted and a YAML-1.1 reader (Ansible, older Ruby) would read it as a bool. Use
  `yaml.NewEncoder` + `SetIndent(2)` to keep v2's layout; pin both behaviours with tests.
- **Fix the dead release branch:** `DefaultsType()` contains `if false { return "release" }`, a leftover from
  Storj's `version.Build.Release` check, so *every* depin binary silently gets dev defaults. Replaced with
  `SetRelease(bool)` that `DefaultsType()` consults.
- **`FindFlagEarly` scans `os.Args`**, which `go test` fills with `-test.*`, making `SetupFlag` (15 depin call
  sites) and `DefaultsFlag` (8) untestable without mutating globals. Add the seam now:
  `FindFlagIn(args []string, name string)`, with `FindFlagEarly` delegating via `os.Args`.
- **`AtomicWriteFile` silently discards its `os.FileMode`** (`atomic.go:11`) — `SaveConfig`'s `0o600` is dead
  code and the file is 0600 only because `os.CreateTemp` happens to be. Honour the mode (`Chmod` before
  rename); the existing permissions test still passes.
- **`ApplicationDir` uses the deprecated `strings.Title`**, suppressed today by paired `//lint:ignore SA1019`
  + `//nolint:staticcheck`. Replace with an inline first-rune `unicode.ToUpper` — avoids both the
  suppression and a `golang.org/x/text` dependency in a config library.
- **Prune dead exports** while renaming is already happening: `SetBoolAnnotation`, `FindConfigDirParam`,
  `FindIdentityDirParam`, `FindDefaultsParam`, `UseDevDefaults`, `UseReleaseDefaults`, and
  `BasicHelpAnnotationName` have zero callers outside `cfgstruct`/`process` in depin. Keep only what is used
  (`IdentityDir` ×29, `ConfDir` ×28, `SetupFlag` ×15, `ConfigVar` ×9, `DefaultsFlag` ×8, `Bind` ×8,
  `UseTestDefaults` ×7, `SetupMode`/`FlagPrefix`/`BindOpt` ×4, `FlagSource`/`AnySource` ×2, `DefaultsType` ×1)
  plus `SetBoolAnnotation` if the README documents it.
- **`DefaultCfgFilename`/`DefaultSecretFilename` stay `const`** (turning them into `var` breaks any consumer
  using them in a const context); the override lives on the options struct. Env prefix stays
  `AIOZ_ENV_PREFIX`-overridable with the `"aioz"` fallback, also settable via option.

Retained deps: `spf13/cobra`, `spf13/pflag`, `spf13/viper`, `spf13/cast`, `zeebo/structs`, `gopkg.in/yaml.v3`.

## Part 2 — lint / CI applied to this repo

Copy `.golangci.yml`, `Makefile`, and `.gitlab-ci.yml` **verbatim from `/home/tuan/work/templates/aioz-stats`**
(golangci-lint pinned `v2.11.4`, `GO_IMAGE: registry:5000/go:1.25`, blocking lint job with checkstyle
artifact, `rules:` on MR + default branch). Two edits: reword the header comments for this repo, and add a
`test` job to `.gitlab-ci.yml` (`stages: [lint, test]`, `go test ./...`) — aioz-stats has none, but this
repo's value is in its tests.

Lint findings the port will hit on the first `make lint`, to fix while writing rather than after:
`misspell locale: US` rejects "behaviour" in the ported `config_test.go:154`; `prealloc` flags `SaveConfig`'s
`var flatKeys []string` and `var lines [][]byte`; `staticcheck` SA1019 on `strings.Title` (removed above);
`gocritic commentedOutCode` on the `if false` branch (removed above).

## Verification

End-to-end first, per the standing preference.

1. **`e2e_test.go` (external package `config_test`) — the flagship.** Real cobra root command with a
   `--config-dir` persistent flag via `SetupFlag`, a `setup` subcommand annotated `{"type":"setup"}`, and a
   `run` subcommand. Config struct exercises a nested struct, `path:"true"` with `default:"$CONFDIR/data"`,
   a `[]string`, a `time.Duration`, and `user`/`setup`/`hidden`/`source:"flag"` fields. In a `t.TempDir()`:
   - `setup` → assert `config.yaml` exists at 0600, defaults commented out, `user:"true"` live,
     `setup`/`hidden`/`source:flag` keys absent, keys sorted, and the `[]string` rendered at the chosen indent
     (this is the yaml.v3 golden).
   - Hand-edit a value, then with a **fresh** command run `run` → assert the struct carries the edited value
     **and** `cmd.Flags().Lookup(key).Changed == true` (guards the propagation side effect `SaveConfig` relies on).
   - `t.Setenv("AIOZ_SERVER_ADDRESS", ...)` → env beats file; then pass `--server.address=` explicitly →
     flag beats env. Both directions, or precedence is not actually tested.
   - `$CONFDIR` expanded to `filepath.Join(tmp, "data")` with `filepath.FromSlash` applied.
   - A `secrets.yaml` merges over `config.yaml`.
   - An unknown YAML key lands in `MissingKeys`; an unparseable value lands in `BrokenKeys`, and with
     `WithFailOnValueError()` returns an error.
   This closes depin's largest test gap — `LoadConfig`/`ViperWithCustomConfig` have no tests there.
2. **`save_test.go`** — port all 9 tests from `depin/pkg/process/config_test.go`. All 9 assertions are
   yaml.v3-safe as written (the fixture binds only string/int/float64, which is luck, not coverage), so add a
   `[]string` golden and a string-valued `"yes"` case to pin the two v2→v3 differences deliberately.
3. **`bind_test.go`** — port `cfgstruct_test.go` (the `int32` panic message naming type/flag/field; the
   supported-types matrix), plus the `getDefault` precedence matrix incl. its "opposite tag without the
   primary" panic, `flagname` regex rejection, `noflag`/`noprefix`/`internal`/`setup` filtering, anonymous vs
   named struct prefixing, array zero-padding (`.00.`, `.01.`), annotations landing on the pflag, and a
   custom `pflag.Value` field.
4. **`case_test.go`** — `snakeCase` table covering both acronym branches: `HTTPServer`→`http_server`,
   `ID`→`id`, `APIKey`→`api_key`, `MaxAttempts`→`max_attempts`, single-char, empty.
5. **`load_test.go`** — decode loop in isolation: used/missing/broken partitioning, the `type: helper`
   logging gate, `Load` called twice on one command, and one regression test per decode bug above.
6. **`paths_test.go`** — `AtomicWriteFile` overwrite, no leftover temp on failure, resulting mode honoured;
   `IsValidSetupDir` missing/empty/has-`config.yaml`; `IsWritable`, which must `t.Skip` when
   `os.Geteuid() == 0` — the CI image runs as root and root writes into a 0500 dir, so the negative case
   only fails in CI.
7. `make lint` zero findings (blocking gate) and `go test ./...` green.
8. **Consumer smoke test:** in a scratch dir outside the repo, a small `main.go` with a `replace` to the
   local module that binds a struct, runs setup, and loads — proves the module is importable and that nothing
   `internal/` leaked into the API.

## Follow-ups (not this change)

- Depin can adopt this module later, but it is **not** a drop-in: `process.Ctx` (29 uses) and
  `process.ExecCustomDebug` (9) stay behind, so depin keeps a thin local `process` owning lifecycle/logging
  that calls into `config.Bind`/`config.Load`/`config.SaveConfig`. State this in the README so the first
  adopter is not surprised.
- Report back to team-playbook: `revive`'s `package-naming` forbids underscores in package names while
  `conventional/naming.md` mandates snake_case package names — the template's traceability table is wrong.
- After approval, write this plan to the vault at
  `Projects/aioz-config/plans/2026-09-04-extract-lint-and-config-package.md` via the hermes CLI, refresh
  `Projects/aioz-config/INDEX.md`, and record durable decisions in `.agent/memory/aioz-config/`.
