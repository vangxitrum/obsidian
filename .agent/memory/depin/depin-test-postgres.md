# depin-test-postgres

testplanet and coord/db tests **skip silently** unless `DEPIN_TEST_POSTGRES` is set.
`go test ./internal/testplanet/` then prints `ok ... 0.2s` — which looks exactly like a
pass. Always confirm with `-v` and look for `SKIP`.

Dedicated test container on this machine is `depin-pgtest`, host port 15445:

    export DEPIN_TEST_POSTGRES='postgresql://depin:depin@localhost:15445/depin_test?sslmode=disable'

**Do not** point it at `coord-db` on :5445 — that is a live coordinator database, and
testplanet creates and drops schemas.

If the container is missing, recreate it WITH a raised connection limit - the stock
`postgres:15` default of 100 connections makes the parallel suite fail ~25 tests in 0.03s
each with `pq: sorry, too many clients already` (planet setup, not the tests):

    docker run -d --name depin-pgtest -e POSTGRES_USER=depin -e POSTGRES_PASSWORD=depin \
      -e POSTGRES_DB=depin_test -p 127.0.0.1:15445:5432 postgres:15 -c max_connections=1000

(Found 2026-09-10: the container had vanished; a default-config recreate produced exactly that.)

The full suite runs ~380s. Start it with `nohup ... &` rather than a foreground Bash
call: it outlives the 120s tool timeout, and a `run_in_background` waiter loop can be
killed independently while the nohup'd suite keeps running to completion.
