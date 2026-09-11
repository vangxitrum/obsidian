---
type: fact
tags: [go, aioz-stats, testing, gotcha]
created: 2026-09-04
agent: main
---

`testcontext.Context.Check(fn func() error)` in `aioz-stats/testcontext` **calls fn immediately** and records the error as a test failure. It is an error assertion, not a deferred-closer registration:

```go
func (c *Context) Check(fn func() error) {
	if err := fn(); err != nil { c.t.Error(err) }
}
```

Writing `ctx.Check(srv.Close)` right after starting a server therefore shuts it down on the spot; the first request gets `connection reset by peer` and everything after it `connection refused`. Use `t.Cleanup(func() { _ = srv.Close() })` for deferred shutdown, and reserve `Check` for a close you actually want to happen and assert now.

`testcontext.New(t)` itself is fine: a `context.Background()` derivative cancelled via `t.Cleanup`. It has no deadline of its own.

Applies to [[service-template-scaffold]].
