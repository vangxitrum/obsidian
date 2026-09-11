# templates — memory index

Go library/scaffold repos under /home/tuan/work/templates.

- [[aioz-common-consolidation]] — aioz-config/logger/stats merged into one aioz-common module; layout, import-path map, history-merge caveats, pinned deps.
- [[aioz-common-depin-extraction]] — 12 domain-free packages moved depin -> aioz-common via git subtree; the errs v1->v2 conversion, the exact history caveat, and what was deliberately left behind.
- [[aioz-common-secret-package]] — new `secret` package: Token/Hash split, the Go 1.24 crypto/rand fact, the two fmt redaction holes (incl. vet skipping Formatter types), and the depin migration notes.
- [[aioz-common-config-secret-guard]] — `secret` struct tag + `CheckSecrets`: a shipped default on a secret panics at bind, an unset one refuses to boot; the devDefault/releaseDefault:"" pairing gotcha.
- [[aioz-common-version-build-stamp]] — `version/` moved top-level + SemVer + monkit series; the git-describe ordering trap, the derived `v0.0.0-dev` format, and what was deliberately not ported from Storj.
- [[aioz-common-redis-rabbitmq-plan]] - planned `redis/` + `rabbitmq/` client packages; the vault-folder mapping gotcha (aioz-common lives under Projects/templates), the hub's reconnect bug, and the amqp091/go-redis traps.
- [[aioz-common-jwt-package]] — `jwt/`: algorithm pinned at construction (alg-confusion attack proved working against a naive verifier), exp required, kid Ring for rotation; and when NOT to use a JWT at all.
