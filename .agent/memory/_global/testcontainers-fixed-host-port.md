# testcontainers: a stop/start test needs a fixed host port

`container.Stop()` then `container.Start()` keeps the same container, but
Docker assigns a **new** host port when the mapping was ephemeral
(`ExposedPorts: []string{"5672/tcp"}`). A reconnect test then measures the
harness rather than the code, because the client is still dialing the old port.

Bind explicitly instead - pick a free port with `net.Listen("tcp",
"127.0.0.1:0")`, close it, and request `"<port>:5672/tcp"`.

Also: the RabbitMQ image's own `guest` user only accepts loopback connections,
so from outside the container it is refused with `ACCESS_REFUSED`, which reads
exactly like a wrong password. Set `RABBITMQ_DEFAULT_USER` /
`RABBITMQ_DEFAULT_PASS`.

Found 2026-09-09 in `aioz-common/rabbitmq`'s tests.

Related: [[rabbitmq4-transient-queues]]
