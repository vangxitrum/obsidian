# cfgstruct bindable types (coord startup panic)

`internal/cfgstruct` binds a **closed set** of leaf field types:
`int, int64, uint, uint64, float64, string, bool, time.Duration, []string`.

Anything else (`int32`, `uint32`, `float32`, `[]int`, named scalar types...) hits the
`default:` arm of the type switch and **panics inside `init()`** of the cmd package -
i.e. before `main()` runs, so *every* role of the binary dies, not just the command
that owns the bad field.

## The 2026-08-06 incident

`coord jobq` was added with `ServerConfig.MaxAttempts int32` (mirroring
`jobqueue.Config.MaxAttempts`, which is int32 because of the proto). Because
`cmd/coord/main.go`'s `init()` binds every command's config, the whole `coord`
binary panicked at startup in *all* containers (api, core, repair, ...):

    panic: invalid field type: int32

The old message had **no flag name and no field name** - only a stack of
`bindConfig` frames. Reading which field it was required decoding the prefix
string length in the stack args (`{0xc0009aefb8, 0x7}` = 7 chars = `"server."`).

## Fixes applied

1. `internal/cfgstruct/cfgstruct.go` - panic now names the flag and the Go field:
   `invalid field type int32 for config flag "server.max-attempts" (field jobq.ServerConfig.MaxAttempts): supported types are ...`
2. `coord/jobq/config.go` - `MaxAttempts` is `int`, narrowed with `int32(...)` at the
   `jobqueue.Config` call site in `coord/jobq/peer.go`.
3. Regression tests: `internal/cfgstruct/cfgstruct_test.go` (panic message names the
   field; all advertised types bind) and `cmd/coord/bind_test.go` - the latter is the
   real guard: merely loading the `main` test package runs `init()`, so any future
   unbindable config field fails `go test ./cmd/coord` instead of crashing production.

## Reusable rule

Config structs must use plain `int`/`int64`; narrow to `int32`/`uint32` at the call
site. The other cmd packages (`worker`, `edgeserver`, `versioncontrol`, `keytool`,
`worker-updater`) still have **no** bind test - the `cmd/coord/bind_test.go` pattern is
~20 lines and worth copying if one of them grows more config.
