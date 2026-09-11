# testcontainers: an unanswered DOCKER_HOST silently falls back to the socket

testcontainers-go (v0.40, `internal/core/docker_host.go`) tries hosts in order:
`tc.host` property, `DOCKER_HOST`, context, `/var/run/docker.sock`, ... Each one
is checked with `docker info`, and a failure is **swallowed** - it moves on to
the next. The winner is cached in a `sync.Once` for the whole test process.

On a GitLab runner that mounts the host's docker socket next to a `docker:dind`
service, a dind that is slow or broken means containers start on the runner's
daemon while `TESTCONTAINERS_HOST_OVERRIDE=docker` sends connections to the
empty dind. Every test then burns its 60s startup timeout:

```
external check: check target: retries: 579 address: docker:32839: get state:
Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.51/containers/.../json": context deadline exceeded
```

Tells: the escaped `%2Fvar%2Frun%2Fdocker.sock` in the URL (client is on the unix
socket, not tcp), and ~580 retries = the dial got ECONNREFUSED every 100ms.

Local repro: `DOCKER_HOST=tcp://127.0.0.1:2375` plus an override pointing at an
idle container's bridge IP.

Fix used in aioz-common (2026-09-10): `requireDocker` compares
`provider.Client().DaemonHost()` to `DOCKER_HOST` and fails in seconds naming
the fallback; the CI job waits for `docker:2375` before `go test`.

Why dind didn't answer (confirmed 2026-09-10, runner `ubuntu-dind-runner`): the
runner is not `privileged`. The dind service log ends at `mount: permission
denied (are you root?)` / `Could not mount /sys/kernel/security`, and dockerd
exits 1 - reproduced locally with `docker run docker:27-dind` minus
`--privileged`. Also note locally: dind listens on 2375 only with
`DOCKER_TLS_CERTDIR=""`, otherwise 2376+TLS.

Decision (2026-09-10): aioz-common's integration job dropped dind and uses the
runner's mounted socket (socket binding) - no DOCKER_HOST, no host override,
Ryuk left on so a cancelled job doesn't leak containers on the shared daemon.
testcontainers reaches containers via the job network gateway; verified by
running the suites in `golang:1.25` with only the socket mounted. The rabbitmq
fixed-port harness retries on "port is already allocated", since another job's
container can hold a port that looks free inside the job container.

Related: [[testcontainers-fixed-host-port]]
