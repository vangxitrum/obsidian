# aioz-depin-wrapper - CI/CD constraints

## CI image: go-ubuntu14.04:1.25-ext (2026-08-24)
The pipeline runs on `registry:5000/go-ubuntu14.04:1.25-ext`, built from `ci/Dockerfile`
in this repo. Ubuntu 14.04 base inherited from `1.23-ext`, Go swapped to **1.25.14**
(checksum-verified from go.dev), plus `curl tar unzip zip lftp` baked in.

Why it exists: the module declares `go 1.25.3`, and `golang.org/x/sync` / `golang.org/x/mod`
both require go >= 1.24.0, so the old `1.23-ext` image (Go 1.23.4) could never build it.
An earlier attempt downgraded the module to 1.23 instead; the user reversed that - bump the
image, not the module.

Rebuild / bump:
```
docker build --provenance=false --sbom=false -f ci/Dockerfile \
  -t 10.0.0.106:5000/go-ubuntu14.04:1.25-ext ci
docker push 10.0.0.106:5000/go-ubuntu14.04:1.25-ext
```
`--provenance=false --sbom=false` matters: the registry is a plain `registry/2.0` and buildx
attestation manifests are not worth the risk there. Registry is `10.0.0.106:5000` from a dev
box, `registry:5000` from inside the runner network - same host, different name.

The image sets `GOTOOLCHAIN=local` on purpose: if go.mod ever outgrows the baked toolchain the
job fails loudly instead of silently downloading another one (which an isolated runner cannot do).
Note `GOTOOLCHAIN=auto` on a dev box hides this class of break - the local base Go here is 1.23.2
and `go version` reports 1.25.x because auto-download kicked in. Verify with `GOTOOLCHAIN=local`.

## GitLab CI gotchas hit in this repo
- A job that gets `rules:` from a YAML anchor (`<<: *anchor`) **discards** the `rules:` coming from
  `extends:` - job-level keys win. The old file had `.standard-rules` silently dead. Use
  `extends: [.setup, .standard-rules]` and keep `rules:` in exactly one place.
- Job-level `when:` cannot coexist with `rules:` - GitLab rejects the whole pipeline config.
- macOS runners are **bash** shell executors. `source ~/.zshrc` returns non-zero when the zshrc
  ends on a false test, which kills the job in 0s with no output. Source profiles under
  `set +e` and set PATH explicitly instead.
- `lftp` was absent from the 1.23 image, so every FTP upload depended on a successful apt-get at
  job time. Baked into 1.25-ext; the before_script apt-get is now a guarded fallback.

## Windows runner is Windows PowerShell 5.1, not pwsh 7
`.gitlab-ci.yml` invokes `powershell -NoProfile -File build.ps1`, i.e. **5.1**. Under
`$ErrorActionPreference = "Stop"`, 5.1 turns ANY stderr output from a native command into a
terminating `NativeCommandError` - even when the command succeeded, and even when stderr is
redirected with `2>$null`. This killed the Windows job at
`git describe --tags --abbrev=0 2>$null` (repo has no tags) and would also have killed
`go build` the moment it printed `go: downloading ...`.

Rule for build.ps1: never call a native binary directly. Use the `Invoke-Native` /
`Invoke-GitOrDefault` helpers at the top of the file, which run the call under
`ErrorActionPreference = 'Continue'` and judge it by `$LASTEXITCODE`.

pwsh 7 does NOT reproduce this (7.3+ changed the behaviour), so a container test passes while
the runner fails. To stress-test helper logic on pwsh 7, set
`$PSNativeCommandUseErrorActionPreference = $true` - strictly harsher than the 5.1 rule.

## YAML: a bare "word: " inside an unquoted script line silently becomes a map
`- command -v go || { echo "ERROR: not found"; exit 1; }` parses as a MAPPING, not a string,
because of the `: ` in `ERROR: `. GitLab then rejects the job with
"before_script config should be a string or a nested array of strings".
`yaml.safe_load` succeeding proves nothing - assert every script entry `isinstance(x, str)`.
Quote every multi-command script line in single quotes.

## Version drift to watch
`build.sh` (unix) and `build.ps1` (Windows) each hardcode the AI CLI version. They drifted
(2.3.5 vs 2.1.17). Both now read `AI_CLI_TAG` / `AI_CLI_VER` env with matching defaults - bump both.

## Known latent issue (not fixed)
FTP artifact prefix comes from `git describe --tags --abbrev=0` in build.sh, build.ps1 and
create_release_zips.sh. Repo has no tags, so every runner resolves `v0.0.0` and it happens to
agree. If tags ever land and a runner fetches them inconsistently, the package stage will look in
a different FTP path than the build stage wrote to.
