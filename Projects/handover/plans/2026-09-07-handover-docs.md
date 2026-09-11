---
type: plan
project: handover
tags: [handover, plan]
created: 2026-09-07
---

# Handover documentation set

## Context

Tuan is leaving / changing role and must hand over work spanning many projects. No handover template exists anywhere: the Obsidian vault (`/home/tuan/personal/tuan/`) has `Projects/*/{INDEX.md,plans,specs,releases,docs}` but zero handover or handoff notes, and `/home/tuan/personal/handover` is an empty directory.

The raw material already exists and is unusually good - 11 vault project folders with dense `INDEX.md` files carrying per-plan status prose, plus 25 `.agent/memory/<project>/` folders holding exactly the tribal knowledge a successor needs (gotchas, traps, systemic bugs, credential leaks). The job is to define a handover template and mechanically project that existing material through it, then flag the gaps only Tuan can fill.

Intended outcome: a self-contained, portable markdown set in `/home/tuan/personal/handover` that a successor can read cold, with no Obsidian required.

**Scope confirmed with user.** In: all 4 core services, all 6 shared-lib/tooling items, and the 3 unowned `~/work` repos. Out: `k6-network`, `ssh-management`, `jira`, `be-skills`, `storj`, `go-livepeer-openpool`, and every personal project.

Also in scope: **scheduled jobs on remote VPS hosts that Tuan checks by hand.** These are an unwritten duty with no artifact anywhere - the local crontab holds only personal `trace` jobs, the systemd timers are personal or OS-default, and nothing in the vault or `.agent/memory` records them. A manually-eyeballed job is the single most fragile thing in a handover: it fails silently the week after the person who watched it leaves. Per the user's choice these are documented per project, not in a central runbook.

## Recon findings that must land in the docs

Discovered while surveying; these are the highest-value handover facts and are easy to lose:

- `~/work/templates/aioz-template` - **2 unpushed commits** on `main`.
- `~/work/go-aioz-media` - **27 uncommitted files** on `feat/support-new-ui` (last commit 2026-08-24). The `depin-workspace/go-aioz-media` checkout is clean, so this is a second working copy holding the only copy of that work.
- `~/work/map/crawler` - 2 dirty files on `feat/setup-weatherpipeline`.
- `~/work/stream/aioz-stream` - 1 dirty file on `feat/new-cdn`.
- `crawler-service` - the v0.0.1 release review verdict was **fix-first: committed OAuth secret in `config.docker.yaml`**; the memory note `agm-crawler-test-coverage` separately records a **committed Mongo credential** and 3 pre-existing test failures.
- `depin` memory records `2026-08-10-worker-initiated-withdrawals` as "Implemented, uncommitted" - the depin repo is clean today, so this needs reconciling: either it landed or it was lost.
- `depin` `2026-08-09-git-release-flow`: Phase 1 (docs) done, **Phase 2 (CI rule changes) not started**.
- `~/work/aioz-ipfs`, `~/work/ipfs`, `~/work/runner` are non-git directories with no vault or memory notes - likely scratch clones, need an explicit "not owned work" verdict rather than silence.
- `~/work/backend` (the path cited by the `backend` memory index) did not resolve during recon; `~/work/backend-develop` (non-git) and `~/work/map` exist. Resolve which is the real MAP checkout.
- **No scheduled work is documented anywhere.** `crontab -l` on this machine returns only the three personal `trace` jobs; the only non-OS systemd timer is `writing-coach-weekly`. Every work cron lives on a remote host and exists solely in Tuan's head.
- `~/.ssh/config` names ~20 hosts that are the candidate cron hosts, and is the best available inventory: `stagging` 157.230.195.37, `hub` 35.213.189.86, `hub-backup` 139.59.103.173, `uplink-prod` 139.59.217.109, `uplink-test` 146.190.95.188, `stream-prod` 167.172.5.171, `skeleton-worker` 178.128.62.128, `monitor` 159.223.90.162, `rpc` 167.71.221.152, `ipfs` 167.172.4.178, `dvault` 139.59.217.252, `a-tunnel`/`b-tunnel`/`c-tunnel` (`{a,b,c}.wnode.cyou`), plus LAN machines.

## Deliverable

A flat markdown set at `/home/tuan/personal/handover`. Plain markdown only - no `[[wikilinks]]`, since the doc must read correctly outside Obsidian. Vault material is referenced as a plain relative path, e.g. `vault: Projects/depin/plans/2026-08-14-coord-resolve-performance.md`.

```
/home/tuan/personal/handover/
  README.md                     index + ownership/criticality table
  HANDOVER-TEMPLATE.md          the reusable template (the thing asked for)
  00-role-overview.md           what the role actually covered; systems map
  01-access-and-secrets.md      access-transfer checklist (names, never values)
  02-loose-ends.md              uncommitted/unpushed/unresolved inventory
  03-cross-cutting.md           house conventions shared by every project
  projects/
    depin.md            backend-map.md      aioz-stream.md    crawler-service.md
    aioz-config.md      aioz-template.md    aioz-stats.md     aioz-depin-wrapper.md
    go-aioz-media.md    team-playbook.md    unowned.md
```

### `HANDOVER-TEMPLATE.md` section order

Ordered so a successor's first 30 seconds and first week are both served.

1. **At a glance** - status (active/paused/shipped/dormant), criticality (P0-P3), repo path + remote + Go module, current branch, suggested next owner.
2. **What it is and why it exists** - 2-4 sentences, no jargon.
3. **Where things live** - repo, vault notes, CI config, registry, deploy target, dashboards, tickets.
4. **How to run it** - prereqs, local dev bring-up, build, test, the 5 commands used daily.
5. **Architecture in brief** - the shape, plus key entry points as `file:line`.
6. **Current state** - shipped / in flight / uncommitted, each with a status word.
7. **Open work** - one line per item, each ending in a concrete *next step*, not a wish.
8. **Traps and tribal knowledge** - the non-obvious things. Sourced from `.agent/memory/<project>/`.
8b. **Scheduled jobs and routine checks** - present only when the project has any. One row per job: what it does, host (ssh alias + IP), schedule, log path, **how to tell it ran correctly**, **what to do when it did not**, escalation. Followed by a short "routine duties" list stating the actual cadence Tuan performs (daily / weekly / on-close) so the successor inherits a calendar, not just a table. A job with no alerting is marked `UNMONITORED` and gets a recommendation for what alert would replace the human eyeball.
9. **Dependencies and blast radius** - what consumes this, what breaks if it stops.
10. **Access and secrets to transfer** - what exists and who must grant it. Never the values.
11. **Risks and landmines** - known-bad things shipped or pending.
12. **First-week checklist for the new owner** - 5-8 concrete actions.
13. **Open questions** - what only Tuan knows and must answer before leaving.

Every unknown is written as a literal `> TODO(tuan): <question>` blockquote so gaps are greppable (`grep -rn 'TODO(tuan)'`) rather than silently absent.

## Implementation

**Step 1 - scaffold.** Create the directory tree and write `HANDOVER-TEMPLATE.md` with the 13 sections, each carrying a one-line instruction comment on what belongs there.

**Step 2 - per-project drafts.** For each of the 11 project docs, gather from four sources and fill the template:
- `Projects/<p>/INDEX.md` in the vault - status prose per plan, already written in past tense with outcomes. Primary source for sections 6 and 7.
- `.agent/memory/<p>/` notes - primary source for section 8, and the only source for it.
- Repo recon: `git` branch/dirty/unpushed state, `README`, `Makefile`, `.gitlab-ci.yml`, `docker-compose.yml`, `go.mod`. Sources sections 1, 3, 4, 5.
- Cross-references between vault plans - sources section 9.

Depth is proportional to surface: `depin` is the large one (25+ plans, specs, component docs, k8s runbooks); `aioz-stats` and `aioz-depin-wrapper` are a page each; `unowned-repos.md` is a single page covering `aioz-ipfs`, `ipfs`, `runner` with a keep/discard verdict per repo.

**Step 2b - scheduled-job inventory.** Nothing on disk records these, so they must be collected from the hosts and from Tuan.

- Enumerate, read-only, over ssh across the candidate hosts listed in the recon section: `crontab -l`, `sudo ls /etc/cron.d/`, `systemctl list-timers --all`, and any `docker compose` services with a scheduler. **This touches remote production machines, so I will confirm the host list before connecting and will run read-only commands only.** Hosts that are unreachable or that Tuan says are not his are recorded as such rather than skipped silently.
- Attribute each discovered job to a project and write it into that project's section 8b.
- Anything found on a host but not attributable to an in-scope project still gets recorded - in `unowned-repos.md`, renamed `unowned.md` to cover both stray repos and stray jobs - because an orphan cron is worse than an orphan repo.
- Separately capture the routine Tuan performs that no crontab shows: the manual daily/weekly eyeball. This comes from him, not the machines, so it lands as `> TODO(tuan):` prompts asking, per project: what do you look at, how often, what does bad look like, and what have you had to fix by hand more than once.

**Step 3 - cross-cutting docs.**
- `03-cross-cutting.md`: the house Go template chain (`aioz-template` composing `aioz-config` + `aioz-logger` + `aioz-stats`), the shared golangci-lint/GitLab CI setup, the `feature/* -> develop -> release-X.Y -> main -> tag` git flow from `depin` `2026-08-09-git-release-flow`, the release + release-review process visible in `depin`/`backend`/`crawler-service`, and the Obsidian-vault + `.agent/memory` documentation workflow itself (a successor inherits the notes and should know how they are maintained).
- `02-loose-ends.md`: the recon table above, one row per repo, each with a required disposition - push / commit / discard / hand to X.
- `01-access-and-secrets.md`: derived from what the projects reference - `gitlab.internal` account and `GOPRIVATE`/ssh-agent setup, container registry, Loki/Prometheus/Grafana, deploy targets and k8s clusters, the depin coordinator identity keys, plus per-service credentials the configs imply. Adds the **ssh host inventory** from `~/.ssh/config` (alias, IP, purpose, whose access grants it) - without it the successor cannot reach a single cron host. Tuan's personal ssh key must not be the sole path to any host; each host gets an explicit "how does the next person get in" line.
- `00-role-overview.md`: the systems map and how the pieces relate, so a reader knows which project doc to open. Ends with a **routine duties calendar** - the consolidated daily/weekly/monthly rhythm gathered in step 2b, each entry pointing at the project doc that explains it. This is the one central view of the scheduled work; the detail stays per project as chosen.
- `README.md`: table of all 11 with status, criticality, a "has routine duties" flag, and a suggested owner column left as `TODO(tuan)`.

**Step 4 - verify.** As described below.

**Step 5 - vault plan write.** Per the global CLAUDE.md rule, once approved, dispatch the hermes CLI to write this plan to `Projects/handover/plans/2026-09-07-handover-docs.md` and add a link in `Projects/handover/INDEX.md`:
`hermes chat -m kr/claude-sonnet-4.5-agentic -q "<instruction>"`. If that route 404s ("No active credentials for provider: kiro", recorded in `_global/hermes-kiro-credentials-dead`), fall back to `cx/gpt-5.6-sol`.

## Verification

1. `grep -rn 'TODO(tuan)' /home/tuan/personal/handover` - returns a reviewable gap list; every one is a question only Tuan can answer.
2. `grep -rn '\[\[' /home/tuan/personal/handover` - must return nothing. No wikilinks leaked; the set is portable.
3. Every repo path named in a doc resolves: a scripted check that each `~/work/...` path in the docs exists on disk.
4. Re-run the git-state sweep and diff it against `02-loose-ends.md` - the inventory must still match reality at handover time.
5. Cold-read test: open `README.md` and follow it to any one project doc; confirm a reader with no context can find the repo, run it locally, and name the next action. `crawler-service` is the sharpest test - it is small, and its landmine (committed secret) must be impossible to miss.
6. Scheduled-job test, the one that matters most since these are undocumented today: pick one job at random from a project's section 8b and, following only what the doc says, ssh to the host, confirm the job exists on the schedule stated, find its log, and determine from the log alone whether the last run succeeded. If any of those four steps needs knowledge not in the doc, the row is incomplete. Also confirm every job is marked either monitored or `UNMONITORED` - no blanks.
7. `git init` the directory and make one commit, so edits during the handover period are tracked and the set can be handed over as a repo.

## Not doing

- No content written into the Obsidian vault except the plan file in step 5.
- No secret values recorded anywhere - `01-access-and-secrets.md` names what exists and who grants it.
- Not rotating the committed OAuth/Mongo credentials in `crawler-service`. That is real remediation work, out of scope for writing docs; it is recorded as a P0 landmine with a named owner instead.
- No changes to any remote host - no crontab edits, no new alerting, no service restarts. Step 2b reads state only. Where a job is `UNMONITORED`, the doc recommends the alert; setting it up is separate work.
- Excluded projects per the user's selection: `k6-network`, `ssh-management`, `jira`, `be-skills`, `storj`, `go-livepeer-openpool`, and all personal projects.

---

## Addendum - as built (2026-09-07)

Delivered at `/home/tuan/personal/handover`, git-committed (2 commits): 13 project docs +
README, 00-role-overview, 01-access-and-secrets, 02-loose-ends, 03-cross-cutting,
HANDOVER-TEMPLATE, and `sweep-hosts.sh`. ~3,460 lines, 200 `TODO(tuan)` open questions.

**Scope changed during execution.** Two projects were added after the initial selection:

- **parser-service** - had been mis-grouped as a personal project; it is the aioz-map service downstream of crawler-service.
- **k6-network** - added because it is running infrastructure, not just notes.

Still excluded: `ssh-management`, `jira`, `be-skills`, `storj`, `go-livepeer-openpool`, and all
personal projects.

**Findings that changed the priorities:**

1. `aioz-stream` has ~127 uncommitted files in a **hidden worktree** (`~/.treehouse/aioz-stream-370b08/2/aioz-stream`) on a branch with no upstream - the entire template alignment, 11 of 14 slice migrations, and ~20 security fixes including two SQL injections and an unauthenticated credential-disclosure endpoint.
2. `k6-network` has **no git remote at all** plus 19 uncommitted files, and its k6 generator containers run on the `stagging` and `uplink-test` hosts rather than dedicated VPS.
3. `go-aioz-media` has an unpushed commit on a branch with no upstream; `aioz-template` has 3 unpushed commits.
4. The OpenSky OAuth credential flagged in the crawler-service v0.0.1 review is **still committed and live** at `config.docker.yaml:253`.
5. All four shared Go libraries, plus `team-playbook` and `aioz-depin-wrapper`, live in a **personal gitlab namespace**.

**Step 2b (remote host cron sweep) was not run by the agent** - the user chose to run
`sweep-hosts.sh` themselves and paste the output. Every project's §8b therefore still carries
`_pending host sweep_` rows and the routine-duties questions.
