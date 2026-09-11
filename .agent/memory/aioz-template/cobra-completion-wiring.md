---
type: fact
tags: [go, cobra, completion, cli, gotcha]
created: 2026-09-04
agent: main
---

Cobra adds the `completion` subcommand on its own, but **every string flag without a registered completion falls back to filename completion** - so `--log.level <tab>` offers the working directory's files. `aioz-template`'s `internal/cmd/completion.go` walks all flags after `Bind` and applies: enums -> their values, `--config-dir`/`--data-dir` -> `ShellCompDirectiveFilterDirs`, `--log.file.path` -> default file completion, everything else -> `ShellCompDirectiveNoFileComp`. Commands are `NoArgs`, so `ValidArgsFunction` returns NoFileComp too.

Enum values live in `internal/config` (`LogLevels()`, `LogFormats()`) as the same maps `ParseLevel`/`ParseFormat` use, with a test asserting every suggested value parses - otherwise suggestions drift from what the binary accepts.

## Two things this surfaced

- **The generated script is keyed on `root.Name()`**, i.e. the first word of `Use`. A constant there (`AppName = "service"`) installs completion for a command nobody runs, since the Makefile names the binary after the module - and gonew rewrites imports but not string literals, so every generated service inherits the mismatch. Fix: `Use: filepath.Base(os.Args[0])`, with a fallback when it ends in `.test`. `AppName` now only names the config directory.
- **`config.DefaultsFlag` panics on an unknown `--defaults`**, read from argv via `FindFlagEarly` before cobra parses. A half-typed word is exactly that, so TAB after `--defaults d` printed a Go stack trace. Guard with an explicit check before calling it; during a completion request (`os.Args[1] == cobra.ShellCompRequestCmd`) rewrite the partial value to a valid one instead of erroring, so TAB still suggests.

## Testing completions

`__complete <args...>` is the hidden command every generated script calls; the last argument is the word being completed, so do not append an extra `""`. The reply ends with `:N`, the `ShellCompDirective` bitmask - `:0` default (file completion), `:4` NoFileComp, `:16` FilterDirs. Driving the real bash script needs `/usr/share/bash-completion/bash_completion` sourced first (`_get_comp_words_by_ref`) and the binary on PATH.

Applies to [[service-template-scaffold]].
