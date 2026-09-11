# aioz-config — memory index

Project: `/home/tuan/work/aioz-config` — reusable Go config-loading package (module `gitlab.internal/tuan.quang.tran/aioz-config`) plus the team golangci-lint/CI setup applied to itself.

- [[config-package-extraction]] — the port of depin's cfgstruct+process into a flat root package: API renames, the 5 bugs fixed rather than ported, and the load-time gotchas (indexed list keys from zeebo/structs, flags sharing storage with struct fields, setup must Load before SaveConfig, embedded structs must be exported).
- [[lint-setup-source]] — copy lint config from `templates/aioz-stats`, NOT from `team-playbook` (whose template has a dangling YAML anchor and a hardcoded depguard path).
- [[aioz-config-embedded-struct-flattening]] — an anonymous struct field is flattened into the parent's flag namespace; use a named field to nest one.
