---
type: fact
tags: [go, docker, gitlab, modules, gotcha]
created: 2026-09-04
agent: main
---

Building a Go image that depends on `gitlab.internal/...` modules needs **three** separate things, and each fails differently:

1. **Discovery is Go's own HTTPS request, not git's.** For a non-vanity host Go fetches `https://gitlab.internal/<path>?go-get=1` to find the repository. `git config url.<ssh>.insteadOf` does *not* apply to it, so on any base image without the internal CA the build fails with `tls: failed to verify certificate: x509: certificate signed by unknown authority`. Fix: `ENV GOINSECURE=gitlab.internal/*`. `go.sum` still pins every module hash, so integrity is unaffected. The host does not hit this because it trusts the internal CA; `registry:5000/go:1.25` probably does too, which is why CI can be green while a local `docker build` is not.
2. **`GOPRIVATE=gitlab.internal/*`** to bypass the proxy and the checksum DB.
3. **Credentials.** `--ssh default` forwards the *agent*, so the key must be loaded with `ssh-add` - a key file that only `~/.ssh/config` references gives `git@gitlab.internal: Permission denied (publickey)`. CI mounts `--secret id=netrc` instead.

A container also does not inherit the host's `/etc/hosts`; `gitlab.internal` (10.0.0.50) needs `--add-host` or working DNS. Internal image refs like `registry:5000/...` only resolve inside the CI network.

Applies to [[service-template-scaffold]].
