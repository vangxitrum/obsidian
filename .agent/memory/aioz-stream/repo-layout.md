---
type: decision
tags: [aioz-stream, layout, deploy, docker]
created: 2026-09-07
agent: main
---

Repository root tidy-up, 2026-09-07, alongside the slice migration
([[slice-migration-state]]).

**Go packages that were sitting at the root moved under `internal/`:**
- `rabbitmq/` -> `internal/utils/rabbitmq/` (it was a library, not a binary,
  despite being called `main.go`)
- `seeds/` -> `internal/seeds/` (its `../debug.env` path became `../../`)
- `templates/` -> `internal/utils/mail/templates/`, now **embedded** with
  `//go:embed`. They were copied into the image and read by relative path, so a
  build that forgot the COPY produced a binary that started fine and failed on
  the first email. `TEMPLATES_DIR` still overrides the embedded set.

**Every deployment asset moved into `deploy/`, grouped by consumer:**

```
deploy/docker/{api,grpc,live,nginx,postgres}.Dockerfile
deploy/nginx/{nginx.conf,redis_lookup.lua}
deploy/postgres/{00_init.sql,entrypoint.sh}
deploy/redis/redis.conf
deploy/observability/{prometheus.yml,promtail-config.yaml,grafana/datasources.yaml}
deploy/mediamtx/aioz-live.yml
docker-compose.yml    (stays at the root: it is the entry point)
```

`COPY` paths inside a Dockerfile are relative to the **build context**, which
is the repo root - so they had to gain the `deploy/` prefix even though the
Dockerfile itself moved into that directory.

**Four references were already broken before the move**, which is what the tidy
-up surfaced:
- compose mounted `./grafana/datasources.yaml`; the file was at the root, so
  Grafana was provisioned with no datasource at all
- compose mounted `./aioz-live.yml` and `./aioz-live2.yml`; the first lived in
  `env-example/`, the second existed nowhere
- `make upload-deploy-resource` scp'd `./00_init.sql`, `./aioz-live.yml` and
  `./pgcat.toml` from the root, where none of them are

That target now copies the whole `deploy/` tree instead of a hand-maintained
list of paths, which is what let the list drift.

**Verify a change here with `docker compose config`** - it resolves every build
context and mount, so a moved file shows up immediately.


**Config is YAML now (2026-09-08), not env syntax.** `internal/config` reads
`./<APP_ENV>.yaml` with `viper.SetConfigType("yaml")`, and the 94
`mapstructure` tags are lowercase (`postgres_host`, not `POSTGRES_HOST`).

**Every deployment's existing environment variables keep working**, because
viper's `BindEnv` uppercases a key to find its variable - so the YAML key
`postgres_host` is still overridden by `POSTGRES_HOST`, with the environment
winning over the file. That is what made the switch safe to do in one step.

**docker compose `env_file:` cannot read YAML** - it only understands
KEY=VALUE. So the split is:
- `app.yaml`, mounted into the Go containers (api, grpc, livestream,
  livestream2), which get `APP_ENV: app` and **no** `env_file`. Leaving the
  env_file on them would have made the YAML pointless, since env wins.
- `app.env`, kept only for the postgres, redis, rabbitmq and grafana images,
  which consume `POSTGRES_USER`, `RABBITMQ_DEFAULT_PASS` and friends at
  startup. The two files describe the same credentials from opposite ends and
  have to be kept in step.

**Seven keys in the example were dead** - `stream_hls_url`, `hls_port`,
`zip_size`, `loki_host`, `loki_port`, `gf_security_admin_user`,
`gf_security_admin_password` - referenced by no Go file, no compose service and
no nginx or grafana config. A test now walks the example and fails on any key
no config field reads, which is what found them.

The missing-required-config error names both forms - `rabbitmq_host
(RABBITMQ_HOST)` - because an operator has one or the other in front of them.

**`AppConfig` is grouped too**, into 21 nested structs - `postgres`, `redis`,
`rabbitmq`, `token`, `chain`, `hls_import`, `live` and so on - so the YAML is
nested rather than a flat list of 94 keys.

**Every leaf carries an `env` tag naming its variable**, and the loader binds
`v.BindEnv(key, tag)` rather than letting viper derive the name from the key.
That is the whole trick: grouping changed every key (`postgres_host` ->
`postgres.host`) without changing a single environment variable, so no
deployment had to be touched. Deriving the name instead would have renamed
`EMAIL_FROM` to `MAIL_FROM` and broken things silently.

`requiredFields`, `missingRequired` and the key walker all recurse into the
groups; a required setting nested in `token` is reported exactly like a
top-level one, as `token.access_private_key (ACCESS_TOKEN_PRIVATE_KEY)`.

The rewrite was only 106 call sites across 8 files - much smaller than the flat
struct suggested, because most code reads a handful of settings.

Two generator traps worth knowing if this is ever regenerated: writing `""` for
an unset key breaks any typed field (`time: invalid duration ""`), so unset
keys are written **commented out** with their env name beside them, the way the
template does; and a group whose leaves are all commented out must have its
header commented too, or viper reads the empty mapping back as a bare key.


**gorm.AutoMigrate is gone; the schema is migrations now (2026-09-08).**
`cmd/http/init.go` had `init := true`, so **every replica ran AutoMigrate over
56 tables on every boot**. On a rolling deploy they raced each other, and a
failure took the service down instead of failing the deploy. The triggers were
worse: applied at startup by reading `DB_TRIGGER_FILE_PATH` off disk, from a
file the image might not even contain.

- `internal/migrations/` - `000001_baseline` (56 tables, 19 indexes) and
  `000002_triggers` (8 triggers, 2 functions, 2 partial unique indexes),
  embedded with `//go:embed`.
- `internal/utils/migrate/` - ported from the template (golang-migrate + pgx).
- `cmd/migrate` - `up`, `down`, `version`, `force`. `make migrate-up` etc.
- A `migrate` stage in `.gitlab-ci.yml` between build and deploy, so a schema
  failure stops the pipeline before the new binary is swapped in. It refuses to
  run against a dirty schema.

**The baseline was generated, not hand-written.** `internal/tools/baselinegen`
(behind a `baselinegen` build tag) runs every `repositories.MustNew*(db, true)`
against a scratch Postgres, then `pg_dump --schema-only` gives the exact schema
the service was already running. Verified by diffing that dump against a
database built by the migrations: **nothing AutoMigrate created is missing**,
and the only extras are the trigger.sql objects, which the old startup path
applied too.

**An existing database is marked, not migrated:** `./migrate force 1`
(`make migrate-baseline`). The baseline is `IF NOT EXISTS` throughout so it is
a no-op if run anyway - a property of the baseline only, and a test enforces
that no later migration uses `IF NOT EXISTS`, where it would hide the drift the
migration exists to correct.

`internal/triggers/trigger.sql` and the `DB_TRIGGER_FILE_PATH` config key were
deleted - migration 000002 is the one source of truth now.

**go.mod is on go 1.25.0**, matching the CI image (`registry:5000/go:1.25`).
golang-migrate forced the move off 1.23 - it pulls x/net, x/crypto, grpc and
protobuf forward, all of which need 1.24 - and 1.25 was taken deliberately so
the language version and the image agree. `GOTOOLCHAIN=auto` fetches it
locally. That enabled `usetesting` and `sloglint`
checks that had been dormant: 29 findings, all mechanical (`os.Chdir` ->
`t.Chdir`, which self-restores so paired cleanups were deleted;
`slog.NewTextHandler(io.Discard, nil)` -> `slog.DiscardHandler`, which is new in
1.24 and is what the template used all along).


**The migrate job runs while the OLD binary is still serving.** Order in
`.gitlab-ci.yml` is build -> migrate -> deploy, and it is enforced with
`needs:` rather than by the stage list alone: `deploy` needs `migrate-job`,
`migrate-job` needs `build-job`. Without that, a future edit to the stage list
could reorder them and the failure would be a new binary querying columns that
do not exist yet.

That window is exactly why **every migration must be N-1 compatible**: between
the migrate job and the rotation, the previous release is running against the
schema just applied. Adding a nullable column is safe; dropping or renaming
one, adding NOT NULL without a default, or adding a unique constraint is not -
those take two releases (expand, then contract).

`migrate-job` carries the **same `rules:` block as `deploy`**, so both derive
`DEPLOY_ENV`/`DEPLOY_PATH` the same way instead of migrate reading them from
the build stage's dotenv. Two sources for one value means the day they disagree
is the day the schema is applied to one database and the binary rotated against
another. It also fails fast on an unset `DEPLOY_PATH`, which would otherwise
migrate in the SSH user's home directory, where there is no config file.


**Deploys are tags-only (2026-09-08).** `migrate-job` and `deploy` run on tags
and nothing else; a branch push lints, tests and builds but never touches a
deployment.

Three things had to change for that to work at all, and each would have failed
silently:
1. **`workflow.rules` had no tag rule**, and ended in `when: never` - so a tag
   push produced *no pipeline*. A tags-only deploy would simply never have run.
2. **`.ssh-target` chose the host by branch.** `$CI_COMMIT_BRANCH` is empty in
   a tag pipeline, so neither test matched and `SSH_HOST` was left unset - an
   ssh to the empty string, a long way from its cause. It now switches on
   `DEPLOY_TARGET`, which the job's rules set, and fails loudly on anything
   else.
3. **lint, test and build-job had to gain a tag rule.** Without it a tag would
   ship unverified, and `deploy` would have had no artifact to deploy.

**Which environment a tag goes to is the tag's shape**, since there is no
branch to ask: `v1.2.3` -> production, `v1.2.3-rc.1` (any semver prerelease)
-> staging, anything else deploys nowhere rather than guessing. That convention
was chosen here, not inherited - it is one regex per job to change.

`build-job`'s branch-based `DEPLOY_ENV`/`DEPLOY_PATH` block and its
`dotenv: build.env` report were deleted: both deploy jobs derive those from the
tag in their own `rules:`, so the dotenv was a second, now-wrong source.


**A `: ` in an unquoted `.gitlab-ci.yml` script line silently becomes a map.**
GitLab rejected `migrate-job` with *"before_script config should be a string or
a nested array of strings up to 10 levels deep"*. The cause was one line:

    - test -n "$DEPLOY_PATH" || { echo "ERROR: DEPLOY_PATH is not set"; exit 1; }

YAML reads the `": "` inside the message as a key/value separator, so that
entry parsed as `{'test -n "$DEPLOY_PATH" || { echo "ERROR': 'DEPLOY_PATH is
not set"; exit 1; }'}` - a mapping where GitLab wants a string. The neighbouring
lines were fine only because they sit inside `- |` literal blocks.

**Use `- |` for any script line containing a colon-space**, quoting is easy to
get wrong with shell inside it.

`internal/tools/ci_lint_test.go` now walks every job's `before_script`/`script`/
`after_script` and fails on an entry that is not a string (a `!reference`
expands to a nested array, which is allowed). Verified by reintroducing the bug
and watching it fail. This class of error is otherwise invisible until the
pipeline runs.


**A bare `.gitignore` pattern silently hid source from CI.** `.gitignore` had a
bare `media` (added by "refactor: change video to media", meant for runtime
media artifacts). A bare pattern matches a path segment at **any depth**, so it
swallowed `internal/app/media/`.

The failure mode is nasty: the 8 files that arrived there by `git mv` stayed
tracked, because **git keeps tracking what it already tracks regardless of
.gitignore** - but the 3 files written fresh (`store.go`, `model.go`,
`chore.go`) were never added. Everything built, vetted and linted locally;
CI cloned a `media` package with no store.go and failed with
`undefined: StreamRepository`, `undefined: MediaCaptionRepository`, ...

Fixed by anchoring: `/media/` and `static/media/` instead of `media`.

**`git status` does not show this** - ignored files are invisible to it - which
is why it survived a whole session of checks. `internal/tools/gitignore_test.go`
now fails on any ignored `.go` file, using
`git ls-files --others --ignored --exclude-standard -- '*.go'`, and reports the
offending rule via `git check-ignore -v`. Verified by re-adding the bare
pattern and watching it fail.

**Other bare patterns in that file remain a latent risk** - `input`, `output`,
`sync`, `examples`, `tests`, `bin`, `tmp`, `playlist_cursor` - any of which
would swallow a future package of that name. The test is the backstop rather
than rewriting them all.


**`.gitignore` is an allowlist now (DePIN style), 2026-09-08.** `*` ignores
everything and each thing that belongs in the repo is named. This inverts the
failure: a new file is missing until it is named, which is loud, instead of a
bare pattern silently swallowing a directory.

Two mechanics of the format, both load-bearing and both learned from depin's
own comments:
1. **`!*/` re-includes every directory.** Without it nothing in a subdirectory
   can be re-included at all, because git never descends into an ignored
   directory.
2. **A `!` rule matches ONE path level** unless it uses `**`. `!deploy/*.yml`
   would not cover `deploy/nginx/nginx.conf`.
Order matters: an ignore rule that must beat an allowlist rule comes after it.

Allowlisted here: `*.go`, `go.mod`/`go.sum`, the two submodule gitlinks,
Makefile/README/.golangci.yml/.gitlab-ci.yml/.air.toml, `docker-compose.yml`,
`deploy/**`, `internal/migrations/*.sql`,
`internal/utils/mail/templates/*.html`, `internal/proto/*.proto`, `docs/**`,
`env-example/**`. The root `app.yaml`/`debug.yaml` need no explicit rule any
more - `*` covers them, which is safer than remembering to add one.

**Verify a change to it by materialising a fresh clone**, not by reading it:
`{ git ls-files; git ls-files --others --exclude-standard; }` into a temp dir,
then `go build ./...`. That is what proves the repo is self-contained.

`internal/tools/embed_test.go` now walks every `//go:embed` directive and fails
if a matched file is ignored - the same trap as the media one but worse, since
an ignored .sql still compiles and just ships a binary with an empty schema.
Note `git check-ignore` says nothing about **tracked** files, so both guards
only catch untracked ones - which is correct, because a tracked file is in the
clone regardless.
