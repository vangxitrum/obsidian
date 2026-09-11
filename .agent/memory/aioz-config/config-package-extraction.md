# aioz-config package extraction

`/home/tuan/work/aioz-config` (module `gitlab.internal/tuan.quang.tran/aioz-config`, go 1.25.0, flat root `package config`) is depin's config system extracted as a reusable library: `internal/cfgstruct` + the config half of `pkg/process` + the needed `internal/fpath` helpers, merged into one package.

## Decisions (user-approved 2026-09-04)

- Faithful port, keeping cobra+viper+pflag as hard deps, but the merged API was cleaned up rather than kept name-identical.
- Lint applied to this repo only; no `templates/` surface.
- Flat root package, matching the `aioz-logger`/`aioz-stats` house style.

## Renames vs depin

- `cfgstruct.Bind` -> `BindFlags`; `process.Bind` keeps `Bind` (the ~30-call-site one).
- `process.LoadConfig` -> `ReadConfigFiles`, freeing `Load` for the new decode entry point extracted out of `process.InitBeforeExecute`.
- `cfgstruct.Prefix` -> `FlagPrefix`.
- Package-level `configs`/`vipers` maps -> `Registry` (+ `New`, `Forget`), with package-level funcs delegating to a default instance.

## Load-time gotchas worth remembering

- **`zeebo/structs` reports unmatched list values as indexed keys** (`peers.0`, `peers.1`), which match no flag name. `lookupFlag` in `load.go` walks back to the parent key when the trailing segment is an index and the parent flag is a `pflag.SliceValue`. Without that, list settings claimed by no struct never load.
- **Flags share storage with the struct fields they were bound to.** A value decoded from the config file therefore moves the flag off its default without ever setting `Changed`; `SaveConfig` keys off `f.Value.String() != f.DefValue`, which is why regeneration keeps operator edits. Asserting `f.Changed` after a struct-claimed load is wrong.
- **A setup command must `Load` before `SaveConfig`**, or regeneration comments out everything the operator had set. depin got this for free from `process.Exec`; a standalone consumer must do it explicitly.
- **Embedded structs must be exported types** - reflection cannot `Interface()` an unexported embedded field, so `struct{ appConfig }` panics with "cannot get interface of field".
- `Load` requires parsed flags (returns `ErrFlagsNotParsed`); it reads `--config-dir` eagerly.

## Bugs fixed rather than ported

1. `f.Changed` was set even when `f.Value.Set()` failed, so an unparseable value got persisted live into the next generated config.
2. Missing keys were read with `vip.GetString` regardless of type, silently emptying list-valued flags.
3. `DefaultsType()` had `if false { return "release" }` - every depin binary ran on dev defaults. Now follows `SetRelease(bool)`.
4. `fpath.AtomicWriteFile` ignored its `os.FileMode`; now honoured via `Chmod` before rename.
5. `fileExists` called `log.Fatalf` (would `os.Exit` the host, and `log` is depguard-denied); now returns an error.

Also deliberately not ported: the package-level `flag.String("debug.trace-out")` (registers on `flag.CommandLine` at import time, so two importers panic with "flag redefined"), `pflag.CommandLine.AddGoFlagSet(flag.CommandLine)` (leaks `test.*` flags into generated configs under `go test`), and the `init()` mutating `cobra.MousetrapHelpText`.

The mutex was also held across the caller-supplied config loader, which deadlocks a non-reentrant `sync.Mutex` if the loader calls back into the package; `Registry.Viper` now builds outside the lock and re-checks under it.

## yaml.v2 -> yaml.v3

Only real difference that bit: v3 defaults to a 4-space indent, so `yaml.NewEncoder` + `SetIndent(2)` is required to keep generated files byte-stable for list values. The claim that v3 stops quoting `yes`/`no`/`on`/`off` is **wrong** - it still quotes them; verified by test.

See [[lint-setup-source]].
