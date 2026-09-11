# amqp091: a channel closing settles pending confirms as NACK (found 2026-09-10)

In amqp091-go v1.14, when a channel shuts down, `confirms.Close()` calls
`deferredConfirmations.Close()`, which does `setAck(false)` on every pending
`DeferredConfirmation`. So `DeferredConfirmation.WaitContext` returns
`(false, nil)` - indistinguishable from a real broker basic.nack.

Consequence for `rabbitmq.Publisher.publishOnce`: a connection or channel that
dies between the publish and its ack is reported as "broker nacked", counted
in `nacked`, and NOT retried, despite `Retries` - the one failure the retry
loop exists for. To tell them apart, check whether the channel closed (e.g.
`ch.IsClosed()` or the NotifyClose signal) before treating `!acked` as a nack.

Other review findings from the same whole-module review (HEAD 2523045):
- `lrucache.Add` deletes the evicted key from the map but leaves its list
  element, so the cache grows past Capacity (capacity 2, 10 Adds -> 9 kept);
  `Add` with Capacity 0 panics (nil Back()).
- `memory.Size.Set` panics with index -1 on letters-only input ("MB").
- `rabbitmq.Consumer.handle` caps every handler at DrainTimeout (30s) even
  outside shutdown, and a DeadlineExceeded result is dead-lettered, not requeued.

Related: [[redis-and-rabbitmq-packages]]
