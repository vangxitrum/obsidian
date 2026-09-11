# RabbitMQ 4 refuses transient non-exclusive queues

Declaring a queue with `durable=false, exclusive=false` against RabbitMQ 4
fails with a **connection-level** `INTERNAL_ERROR (541)`:

```
Feature `transient_nonexcl_queues` is deprecated.
By default, this feature is not permitted anymore.
```

It kills the whole connection, not just the channel. Any client that
re-declares its topology on reconnect (as `aioz-common/rabbitmq` does, by
design) turns this into an endless connect/declare/die loop that reads like a
network fault.

Found 2026-09-09 while writing the integration tests for
`aioz-common/rabbitmq`; the fix is `Durable: true` on every queue.

Related: [[testcontainers-fixed-host-port]]
