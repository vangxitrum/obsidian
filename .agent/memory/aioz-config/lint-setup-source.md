# Where the team lint setup actually comes from

`/home/tuan/work/team-playbook/templates/golangci-lint/` holds the *templates* (`.golangci.yml`, `lint.gitlab-ci.yml`, `Makefile.snippet`), but they are not directly usable: `lint.gitlab-ci.yml` ends with a dangling `<<: *on_code_change` YAML anchor that is defined nowhere, and `.golangci.yml`'s depguard rule hardcodes `!**/internal/utils/log/**` (aioz-map/backend's layout).

**Copy from `/home/tuan/work/templates/aioz-stats` instead.** It already adopted the templates and fixed both: real `rules:` (MR event + default branch) in place of the anchor, depguard exclusion dropped, `test` target added to the Makefile, `go 1.25.0`. Pins: golangci-lint `v2.11.4`, image `registry:5000/go:1.25`.

`/home/tuan/work/templates/` also holds `aioz-logger` (module `gitlab.internal/tuan.quang.tran/aioz-log`, flat root package - the house style for shared libs) and `aioz-template`. Note there is *also* a stale clone at `/home/tuan/work/aioz-logger`.

The playbook's traceability table claims `revive`'s `package-naming` enforces snake_case package names; it does the opposite (forbids underscores), contradicting `conventional/naming.md`. Worth reporting upstream.

Applies to [[config-package-extraction]].
