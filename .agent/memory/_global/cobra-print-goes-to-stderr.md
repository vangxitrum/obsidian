---
type: fact
tags: [go, cobra, cli]
created: 2026-09-10
agent: main
---

cobra's `cmd.Print` / `cmd.Println` / `cmd.Printf` write to `OutOrStderr()`. With no `SetOut`, output goes to **stderr**, so `mybin version | grep` reads nothing. For real command output, use `fmt.Fprint(cmd.OutOrStdout(), ...)`.

A test that points `SetOut` and `SetErr` at the same buffer hides the bug. Use separate buffers and assert stderr is empty.

Hit in aioz-stream, where `make verify-stamp` failed on every build: [[single-binary-cli]].
