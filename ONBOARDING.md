# Onboarding: the Sorbit coder harness

Read this first. It is the single self-contained description of the system: what
the pieces are, how a goal becomes a pull request, what is guaranteed and what
only looks guaranteed, where every file lives, and which rules exist because
breaking them already cost us something.

It is written to be sufficient. You should not need to SSH anywhere to answer a
question about how the harness works. When you do need depth — measurements,
build history, the reasoning behind a choice — the other docs are indexed at the
[end](#where-to-go-deeper).

Current as of **2026-09-28**.

---

## 1. What this system is

An AI coding pipeline with one human checkpoint at the end.

A person writes a **goal** as an issue on a self-hosted board. A cloud planner
splits it into **units**. A local code model edits one unit at a time on a GPU
box, with tests and checks after every unit. When the whole integration branch
is green it is pushed to GitHub as a pull request, and a human reviews and
merges. Nothing merges automatically, and no agent holds a credential that could
merge it.

Two design commitments explain most of the structure:

1. **The model that writes code has no network, no credentials and no sudo.**
   It runs as an unprivileged local account on a box with no GitHub key. Its
   output leaves that box only as a branch bundle, pushed by a separate service.
2. **Every gate is honest about its scope.** "Green" is a narrow claim (§6). The
   final human diff review is load-bearing, not ceremonial — a model has already
   once passed every check while silently deleting twelve lines of design notes.

---

## 2. Vocabulary

| Term | Meaning |
|---|---|
| **goal** | The issue a human writes. Parent of everything else. |
| **unit** | One child issue: normally one file, with exact instructions. The atom of work. |
| **integration branch** | `agent/goal-<slug>`. Every unit's commit fast-forwards onto it. Cut from `prod-testing`. |
| **fan-out** | Creating the integration branch + one issue per unit + a verify issue blocked by all of them. |
| **fan-in** | A green unit's commit advancing the integration branch. |
| **verify** | The end-of-goal run of the whole suite against the integration branch. |
| **loop-back** | Verify going red and creating scoped fix units, then re-running itself. |
| **strike** | One failed attempt on a unit. Three strikes escalate. |
| **escalation** | A thrice-failed unit handed to Sentinel, which rewrites the task. |
| **base** | The commit/branch a job starts from. A *resolved* base and an *explicitly requested* one are different things (§7). |
| **manifest** | Per-repo defaults on the executor: `/home/agent/etc/repos/<repo>.json`. |

---

## 3. The two machines

```
┌─ MINI PC · charlie-Precision-WorkStation-T3500 ──────────────────────┐
│  192.168.1.121 · tailnet 100.114.50.113                             │
│  Linux Mint 22.1 / Ubuntu 24.04 · 4 cores · 7 GB · Node 24.20.0     │
│                                                                      │
│  Paperclip :3100 (loopback)   the board + agent runtime             │
│  embedded Postgres :54329                                            │
│  paperclip-gate :3101         bearer-token-only proxy to :3100      │
│  deepseek-meter :4100         holds the DeepSeek key, counts spend  │
│  infra-reader   :4101         read-only GitHub/Neon/GCP + publish   │
│  z440-tunnel                  ssh -L 8080 → the Z440's model server │
└──────────────────────────────┬───────────────────────────────────────┘
                               │  direct cable or home wifi, SSH (not Tailscale)
                               │  three keys, three forced commands
┌──────────────────────────────┴───────────────────────────────────────┐
│  Z440 · jack-1-0 · 192.168.1.199 (static) · LAN ONLY · ufw: 22/tcp  │
│  Node 20.20.2 · 231 GB disk · Radeon, 17.1 GB VRAM                  │
│                                                                      │
│  llama.service   :8080   Qwen3-Coder-30B-A3B · Q4_K_XL · 64K        │
│  litellm.service :4000   gateway, alias qwen3-coder                  │
│  user `agent` (uid 1001, NO sudo, no GitHub key): jobd, verifyd,     │
│                          syncd, aider, the repos, the worktrees      │
└──────────────────────────────────────────────────────────────────────┘
```

The mini PC is the box engineers reach, over Tailscale, as the shared `charlie`
account. The Z440 is reachable only from the mini PC, by SSH over **either of two paths**, same
address both ways:

- **Direct ethernet cable** (`enp3s0` ↔ `eno1`; used from 2026-10-16, no router on
  that link). The mini PC holds a persistent `/32` host route to `192.168.1.199` on
  `enp3s0` (NetworkManager profile "Wired connection 1"); it is only live while the
  cable has carrier.
- **The home LAN over wifi** (mini PC `wlp2s0`, `192.168.1.121/24`; in use from
  2026-11-04, when the cable was removed). With no carrier on `enp3s0` the `/32`
  route is gone and `192.168.1.199` is reached through the ordinary `/24` on `wlp2s0`.

Nothing in the harness changes between the two: `~/.ssh/config`, the keys and the
address are the same. To see which path is in use, `ip route get 192.168.1.199` shows
`dev enp3s0` (cable) or `dev wlp2s0` (wifi). If `ssh z440` says *No route to host*
with the cable plugged in, check `enp3s0` has carrier and that route. On wifi the link
is shared with the rest of the LAN, so latency is less predictable than on the cable.
Not verified: a full goal run over wifi (only `ssh z440` login was tested on 2026-11-04).

**Accounts that matter**

| Box | Account | Powers |
|---|---|---|
| Mini PC | `charlie` | passwordless sudo; shared by all engineers; `ssh z440` as `jack` |
| Mini PC | `glorp` | system user, no sudo, `no_new_privs`, uid-firewalled. Runs Glorp and Sentinel. |
| Mini PC | `dsmeter` | owns the DeepSeek key |
| Z440 | `jack` | owner, passwordless sudo. No GitHub Actions runners (removed 2026-10-16; CI runs on GitHub-hosted runners). |
| Z440 | `agent` | uid 1001, **no sudo**, home 0750, no GitHub credential. Everything the model writes happens here. |

**SSH channels from the mini PC** — each key is pinned to one forced command, so
the channel *is* the API:

| Alias | Key | Lands on |
|---|---|---|
| `z440` | `~/.ssh/z440_key` | full shell as `jack` |
| `z440-job` | `~/.ssh/z440_jobd` | `/home/agent/bin/jobd` |
| `z440-verify` | `~/.ssh/z440_verify` | `/home/agent/bin/verifyd` |
| (root, sync) | `/etc/z440-sync/key` | `/home/agent/bin/syncd` |

---

## 4. The agents

Five agents on the Paperclip board, company **Sorbit**
(`ddeaf594-a0a2-4847-bd99-db9b2866cf0d`).

| Agent | id prefix | Adapter | Brain | Wakes on |
|---|---|---|---|---|
| **Glorp** | `c0e87781…` | `claude_local` | DeepSeek V4.1-Flash (`deepseek-flash`), metered | assignment, comment |
| **Sentinel** | `0ed506f6…` | `claude_local` | same launcher + model as Glorp | assignment |
| **Zed** | `03887377…` | `process` → `~/harness/bin/z440-exec` | Qwen3-Coder on the Z440 | **assignment only** |
| **Verity** | `dd57e84b…` | `process` → `~/harness/bin/z440-verify` | none (runs checks) | assignment, dependency wakeup |
| **Database Operator** | `1a33e10a…` | `process` → `~/harness/db-operator/db_operator.py` | none (fixed scripts) | assignment |

**Glorp** plans. It reads the repo, decides the smallest set of changes, writes a
plan file, and runs `goal-fanout`. It never writes application code. Limits:
1800 s, 200 turns, 2 concurrent runs.

**Zed** codes. It claims one assigned unit, turns the description into a job,
ships it over `ssh z440-job`, and writes the result back. One at a time —
there is one GPU, held under `flock`. Timeout 5400 s, above jobd's 3600 s aider
timeout.

**Verity** checks. It runs the suite against the integration branch, drives
loop-back on red, and on green publishes the PR and hands the goal to a human.
It takes no GPU lock, so it can run alongside Zed. A green verify takes ~3 s.

**Sentinel** reviews. When a unit fails three times, Sentinel reads the three
failure reports and diagnoses them, then either rewrites the unit's description
and resets its strike count, or blocks it for a human. One pass per unit.

**Database Operator** applies reviewed, *named* live-DB runbooks
(`Database-Operation: <runbook>` + `Apply: true`). It never turns issue text
into SQL. Not part of the coding flow — leave it alone.

**Zed's timer wakeup is deliberately off.** Every wakeup can cost a real GPU
job. Work reaches Zed by assignment, never by a clock.

---

## 5. The lifecycle of a goal

```
human writes goal, assigns Glorp
   │
   ├─ Glorp reads the repo (read-only mirror), writes plan-*.md
   ├─ Glorp may comment a QUESTION and stop  ──▶ human answers in a comment
   │
   └─ goal-fanout <goal-id> <plan-file>
        ├─ create-base → integration branch agent/goal-<slug> off prod-testing
        ├─ one child issue per unit, assigned Zed, carrying Repo:/Files:/Base:
        ├─ one verify child, assigned Verity, BLOCKED BY EVERY UNIT
        └─ the goal itself parked `blocked` by its verify issue
   │
   ▼  units run strictly one at a time
 unit todo → in_progress → jobd → aider → Qwen edits → commit
        ├─ per-unit gates: coverage gate, unit suite, flutter analyze (Dart)
        ├─ PASS → fan-in onto the integration branch → unit `done`
        └─ FAIL → strike. Retry monitor scheduled (~2 s), feedback appended
                  to the task. 3 strikes → Sentinel → rewrite + reset → 3 more
                  → still failing → `blocked`
   │
   ▼  last unit closes → dependency wakeup fires the verify issue
 Verity runs the whole suite + analyze + syntax checks on the branch
        ├─ RED  → up to 4 scoped fix units for Zed, verify `blocked` on them,
        │         re-verify when they close. Max 2 rounds → goal `blocked`.
        └─ GREEN→ clear the goal's blockers
                → verifyd `export` (a branch bundle, no credential on the Z440)
                → POST infra-reader /v1/github/publish
                → push agent/goal-* + open PR into prod-testing
                → goal `in_review`, human assignee, agent stood down
                → close the verify issue LAST
   │
   ▼
 human reads the diff and merges the PR. This is the only human touchpoint.
```

A 2-unit goal has gone from assignment to `in_review` in about 2.5 minutes. A
small unit is 20 s–3 min. Units never parallelise: a 10-unit goal costs 10× one
unit.

### Status transitions worth memorising

- A green unit ends **`done`**, not `in_review`. Only `done` blockers resolve
  dependents (`services/issues.ts:1307`), so an `in_review` unit would stall the
  verify issue forever and need a human to close it.
- The **order of the green handback is load-bearing**: clear blockers → hand the
  goal to the human → *then* close verify. Paperclip refuses an agent's status
  change on a `blocked` issue while a blocker is unresolved (409, judged on the
  pre-write state), and closing verify first would fire a dependency wakeup at
  Glorp.
- `in_review` + a human assignee must be **one write** that also sets
  `assigneeAgentId: null`. Two assignees is refused; `in_review` with no owner is
  refused (422).

---

## 6. What "green" actually means

Green means, and means only:

- the `cloud-run` **mocha unit suite** passes, with `SKIP_TESTCONTAINERS=1`
  (511 tests as of SOR-82);
- every **touched Dart file** passes `flutter analyze` — errors fatal,
  informational lints not;
- touched **JS / HTML / CSS / SQL** files pass a **syntax-only** check.

It does **not** mean:

| Not covered | Why |
|---|---|
| Integration, Postgres, live suites | They need docker; `docker` is root-equivalent on the Z440 and would undo the `agent` account's no-sudo isolation. Deliberate. |
| `flutter test` | Flutter 3.47.4 is installed and 115 `*_test.dart` files exist. They do not run. |
| Comments, docs, design rationale | No assertion reads them. A deletion sails through. |
| Runtime behaviour, rendering | Nothing executes the app. |
| Anything wrong-but-passing | The suite only knows what it asserts. |

Every verify report, green or red, carries this scope statement in its own words.
Do not restate it as a full-suite pass.

**So always read the diff.** The end-of-goal human review is the only place a
content regression is caught, and it is reading a diff, not a test result.

---

## 7. The contracts

### Issue description fields (read by `z440-exec`)

```
Repo:   sorbit                    # must exist under /home/agent/repos
Files:  path/one.js path/two.js   # the files aider may edit — see below
Tests:  <command>                 # optional; the manifest supplies Sorbit's
Base:   agent/goal-<slug>         # integration branch; goal-fanout writes this
```

Everything else in the description becomes the task text sent to the model.
**Comment text never reaches the model** — guidance must go in the description.

`Files:` is what makes a large repo tractable. Aider's repo map is capped at
2048 tokens against a 64K window; on Sorbit it will not reliably find anything.
A scoped Sorbit edit sends ~3.2k tokens instead of tens of thousands, and `jobd`
passes `--map-tokens 0` whenever `Files:` is present.

### `jobd` — the job channel

`ssh z440-job < job.json > result.json` is the entire surface.

In: `{repo, task, files, tests?, base?, op?}` · Out: `{id, branch, exit, diff, log}`

Exit codes you will see: **0** pass · **6** coverage gate (under-applied) ·
**75** the base moved underneath the job, no ref updated · non-zero otherwise.

`op: create-base` cuts a new integration branch and deliberately does **not**
resolve its own base.

### `verifyd`

In: `{repo, base, suite?}` · Out: `{id, repo, base, sha, suite_cwd, exit, log}`.
Also serves `export`, a bundle of `agent/goal-*` commits beyond the manifest
base, which is how work leaves a box that holds no GitHub credential.

### The repo manifest — `/home/agent/etc/repos/<repo>.json`

Resolution order is **job value → manifest → built-in default**. A missing
manifest degrades to exactly the pre-manifest behaviour, which is the regression
guard.

Sorbit's carries `base` (`prod-testing`), `tests` (the `SKIP_TESTCONTAINERS=1`
target), `setup`, `node_modules`, `suite_cwd`, `max_file_bytes` (250000),
`dart_analyze`, `syntax_checks`.

The manifest stays marked `UNREVIEWED` until a human clears it. `repo-add` runs
three secret-scanning nets itself and prints `HIT` or `(clean)`, but does not get
to decide that a repo is safe to expose to a model.

### Base resolution: two values, not one

`jobd` keeps **`BASE`** (the resolved start point: job → manifest → HEAD) apart
from **`base_job`** (the explicitly requested integration branch). **Only
`base_job` is ever fast-forwarded.** Conflating them would fast-forward
`prod-testing` to unreviewed model output on every green job.

It also refuses to advance the checked-out branch, and diffs against a *recorded
commit* (`START=$(git rev-parse "$BASE")`) rather than the base *name*, because
fan-in moves the name — a name diff reports empty for exactly the jobs that
succeeded.

### Marker comments (machine-readable, exact strings)

| Marker | Meaning |
|---|---|
| `**z440 retry cycle reset**` | Resets a unit's three strikes. Sentinel writes it; a human can too. |
| `**z440 sentinel escalation**` | Posted when a thrice-failed unit is handed to Sentinel. |
| `**z440 verify failed**` | Verity's failure report. Matched at the **start** of the body, by the authoring agent only. |
| `z440-loopback-round: <files>` | Durable round counter, so loop-back survives a restart. The run itself holds no state. |

### Environment switches

| Var | Where | Effect |
|---|---|---|
| `Z440_DEFAULT_REPO=sorbit` | Zed, Verity | issues with no `Repo:` target Sorbit. Drop it and make `Repo:` mandatory as soon as there is a second repo. |
| `Z440_LOOPBACK=1` | Verity | red verify creates fix units |
| `Z440_MAX_ROUNDS=2`, `Z440_MAX_FIX_ISSUES=4` | Verity | loop-back caps |
| `Z440_PUBLISH=0` | Verity | turns off push + PR |
| `Z440_REVIEWER_USER_ID` | Zed, Verity | handback assignee, default `local-board` |
| `Z440_EXEC_AGENT_ID` | Verity | which agent fix units go to (Zed) |
| `HEARTBEAT_SCHEDULER_INTERVAL_MS=10000` | systemd drop-in | the monitor/heartbeat sweep. 10000 is the enforced floor. |

---

## 8. Failure paths

### Per-unit: three strikes

A failed attempt posts a failure report, appends the failure to the task as
retry feedback, and schedules an **issue-bound retry monitor** ~2 s out. The
monitor fires on Paperclip's heartbeat sweep (now 10 s), so expect a retry to
wake in ~2–12 s.

Strike counting is narrow on purpose: a marker must appear at the **start** of a
comment body and the comment must be **the agent's own**. Sentinel is an LLM
writing prose *about* failures — counting its diagnosis as a strike would
silently spend the one cycle its rewrite was meant to buy. Unknown authorship
still counts, so a miscount can only stop early, never retry on the GPU forever.

Three strikes → Sentinel (§4) → rewrite + reset → three more → `blocked`.
Infrastructure errors (Z440 unreachable) never escalate.

### Per-goal: loop-back

Red verify parses the failing test files out of mocha's spec reporter — a
**heuristic**, documented as one: it reads stack frames
(`at Context.<anonymous> (test/unit/foo.test.js:31:19)`) and makes them
repo-relative via `suite_cwd`. It finds the file a failure was *raised* in, not
the source file that broke it. Analyzer errors are parsed too, error-severity
first.

Caps: 2 rounds, 4 fix issues per round, a file dispatched in 2 rounds is
excluded from a third and named in the comment. No parseable file → left `todo`
for a human rather than guessing. Worst case is 8 fix units, then `blocked`.

### Reading a `blocked` issue

`blocked` always means a human decision is wanted. Read Sentinel's diagnosis and
the failure comments, then re-scope or fix by hand. It does not mean the harness
is broken.

---

## 9. Publishing, and why no agent can merge

On green, Verity asks verifyd for an `export` bundle and posts it to
`infra-reader`'s `/v1/github/publish`. That route takes a **separate publish
key** (`/etc/infra-reader/publish-key`, copy at
`~charlie/.config/infra-publish/key`). Glorp can read neither, and Glorp's own
key gets a 401.

The publisher:

- pushes **only** `refs/heads/agent/goal-*`;
- **never** force-pushes, never deletes;
- opens or reuses a PR into **`prod-testing`** only;
- **refuses any branch touching `.github/`** — PR workflows run with repo
  secrets;
- refuses to push at all while its local copy of `prod`, `pre-testing`,
  `prod-testing` or `master` differs from the remote.

Those last rules live in a **root-owned pre-push hook**
(`/usr/local/lib/infra-reader/hooks/pre-push`), passed as `core.hooksPath` on
the push itself, so the service user can neither edit nor skip it. The publisher
refuses to push if the hook is missing. This stands in for GitHub branch
protection, which needs a paid plan.

The guard covers *pushes*. The App key could still change those branches through
the GitHub API, so it stays readable by root and the `infrareader` service only.

A failed publish never blocks the handback — the goal still reaches the human,
and the comment says why the PR is missing.

---

## 10. Trust boundaries

Each boundary exists because the thing on the other side of it is untrusted for
a specific, stated reason.

| Boundary | Keeps out | Because |
|---|---|---|
| `agent` has no sudo, no GitHub key, no internet | model-written code | it runs code a 3B-active model wrote; a token there is reachable by it |
| Z440 clone, not `jack`'s checkout | the `agent` account | jack's tree holds a live Firebase **admin** credential, gitignored and untracked. `git clone` left it behind; sharing the directory would have handed it to model-written code. |
| `glorp` system user + uid firewall | Glorp/Sentinel | Claude Code with skipped permissions, running as `charlie` with passwordless sudo, is one `sudo cat` from the DeepSeek key or a raised cap. glorp may reach only :4100, :3101, :4101 and the Z440's ssh. No internet, no IPv6. |
| `paperclip-gate` :3101 | any local process | Paperclip runs `local_trusted`: **any** loopback request without a bearer token is the instance-admin board, which can rewrite a `process` agent's command — and that command runs as `charlie`. The gate forwards only token-bearing calls. |
| `deepseek-meter` as `dsmeter` | everyone | the key lives in exactly one place Glorp cannot reach (`/etc/deepseek-meter/key`, 0600). Caps are root-owned. |
| forced-command SSH keys | the job channel | `jobd`/`verifyd`/`syncd` are the whole API; there is no shell on those keys. |
| root-owned pre-push hook | the publisher itself | see §9 |
| `infra-reader` | Glorp | three bounded read operations, separate read-only GitHub/Neon/GCP identities, credentials never in Glorp's environment |

**Still open by choice:** Paperclip stays `local_trusted`, so any *other* local
user or service on the mini PC still gets board access, and there is no
per-person audit trail. Switching to `authenticated` would close that at the
cost of a login — and would need the gate/isolation re-checked.

**What Glorp can read** — `infra-read sync` (refreshes the read-only mirror at
`/var/lib/infra-reader/Sorbit`), `infra-read neon '<single SELECT/WITH>'` (forced
read-only transaction, row/time/output caps), `infra-read gcp logs --filter … `
(fixed viewer commands). No arbitrary `gcloud`, no IAM, no secret values. Audit:
`journalctl -u infra-reader.service`.

---

## 11. Cost and quota control

Glorp and Sentinel run on DeepSeek V4.1-Flash through the meter. Caps:
**$1/hour, $3/day, $30/month** (rolling hour, rolling 24 h, calendar month UTC),
in `/etc/deepseek-meter/config.json` — root-owned, `systemctl reload` to apply.

A typical goal costs **2–4¢**. September to date: $0.87 over 911 requests, ~98%
cache hit rate.

Four layers, outermost last: Paperclip run limits (1800 s / 200 turns) catch one
looping run · meter caps catch runaway across runs and mid-run · the quota guard
(`~/harness/bin/claude-quota-guard`, 5-min timer) pauses metered agents while a
cap is closed and resumes only the ones it paused · DeepSeek's prepaid balance is
the last resort.

Two things to know:

- **Paperclip's `costUsd` is wrong for these agents.** Claude Code prices runs
  from its own Claude table — it reported $0.63 for a run that cost $0.01. The
  meter's ledger (`/var/lib/deepseek-meter/ledger.jsonl`, one line per request,
  attributed by run id) is the source of truth. Paperclip's *token counts* are
  right and can cross-check it.
- **Paperclip's budgets cannot cap subscription work at all.**
  `normalizeBilledCostCents()` returns 0 for `billingType:
  "subscription_included"`, so a `claude_local` agent on a subscription never
  accrues spend and no cents budget can fire. Hence the external guard.

`~/harness/bin/glorp-brain deepseek|claude` switches Glorp's brain; the Claude
config snapshot is kept at `~/harness/state/glorp-config.claude.json`.

---

## 12. Where everything lives

### Mini PC

| Path | What |
|---|---|
| `~/harness/bin/z440-exec` | Zed's bridge. `--selftest` (API+SSH), `--dry-run` (prints job JSON). |
| `~/harness/bin/z440-verify` | Verity's bridge: verify, loop-back, publish, handback |
| `~/harness/bin/goal-fanout` | plan file → integration branch + units + verify issue. Refuses >12 units without `--force`. |
| `~/harness/bin/glorp-brain` | switch Glorp between DeepSeek and Claude |
| `~/harness/bin/z440-stale-sweeper` | wakes Zed for a unit stranded `in_progress` with no run. 5-min user timer, deployed 2026-09-30, log `~/harness/logs/stale-sweeper.log` (see `FANOUT.md`). |
| `~/harness/bin/claude-quota-guard` | + `.timer`, every 5 min |
| `~/harness/state/glorp-AGENTS.md` | **Glorp's instructions.** Scope discipline lives here. |
| `~/harness/state/sentinel-AGENTS.md` | Sentinel's procedure |
| `~/harness/logs/quota-guard.log` | |
| `~/harness/db-operator/README.md` | Database Operator + its runbooks |
| `~/harness/z440-sync/` | sources + installer for the base-sync path |
| `~/harness/deepseek-meter/`, `~/harness/glorp-user/` | sources for the installed services |
| `~/.paperclip` | the managed Paperclip install (0700). **Reference only.** |
| `~/.config/systemd/user/paperclipai.service.d/10-heartbeat-interval.conf` | the 10 s sweep drop-in |
| `/usr/local/bin/llama-gpu`, `/etc/llama/gpu-disabled` | Z440: the model on/off toggle and its persistent flag (see Z440.md) |
| `/etc/deepseek-meter/{key,config.json}` | key (dsmeter, 0600) and caps (root) |
| `/etc/infra-reader/publish-key` | the one write credential |
| `/opt/glorp/` | root-owned copies of Glorp's tools. `claude` pinned at 2.1.268. |

`/opt/glorp` is refreshed with `sudo glorp-sync-tools`, and the Claude pin moves
only with `sudo GLORP_CLAUDE_VERSION=x.y.z glorp-sync-tools`. **After editing
`~/harness/bin/goal-fanout`, run `glorp-sync-tools`** — Glorp uses the copy.

### Z440

| Path | What |
|---|---|
| `/home/agent/bin/jobd` | the per-unit executor |
| `/home/agent/bin/verifyd` | end-of-goal suite; sweeps dead worktrees |
| `/home/agent/bin/syncd` | fast-forward-only base sync from the mirror |
| `/home/agent/bin/repo-add` | clone + survey + secret nets + manifest stub |
| `/home/agent/bin/agent-prune` | nightly: worktrees >7 d, **merged** `agent/*` branches, logs >30 d |
| `/home/agent/etc/repos/*.json` | manifests, outside every working tree |
| `/home/agent/etc/CONVENTIONS.md` | wired into `.aider.conf.yml` as `read:`, so it is in the context of **every** edit |
| `/home/agent/repos/{sorbit,scratch}` | the checkouts the agents share |
| `/home/agent/work/`, `/home/agent/verify/` | job and verify worktrees. Both should be empty at rest. |
| `/home/agent/logs/<id>.log` | per-job logs |
| `/home/agent/.aider.conf.yml` | `edit-format: diff`, `map-tokens: 2048`, `gitignore: false`, `yes-always`, `read: CONVENTIONS.md`, `lint-cmd: sql: check-sql.js` |
| `/home/agent/.aider.model.metadata.json` | `max_input_tokens: 65536` — **must track `-c`** |
| `/home/agent/.aider.model.settings.yml` | `use_temperature: false` |
| `/home/agent/tools/` | the syntax checkers |
| `/etc/llama/api-keys`, `/etc/litellm/env` | 0640 root:jack |

Backup convention: every in-place change leaves a sibling named
`<file>.bak-<what-it-was-before>` (`jobd.bak-preemptycontent`,
`z440-exec.bak-presentinel`, …). That file is the record of when a change
landed.

### Model tuning (do not re-litigate)

`Q4_K_XL` · `-c 65536` · `--n-cpu-moe 18` · `--temp 0.7 --top-p 0.8 --top-k 20
--repeat-penalty 1.05 --min-p 0`, all set **server-side in `llama.service`**.

- **Sampling lives in one place.** Aider has no preset for
  `openai/qwen3-coder`, so it sent `temperature: 0` on every request until
  2026-09-18 — every unit decoded greedily, and a retry differed from the failed
  attempt only by the appended feedback. `use_temperature: false` fixes it.
- **Do not buy context by dropping the quant.** Q4_K_XL is the floor for a
  30B-A3B; the failure mode is syntactically valid, subtly wrong code — the
  worst possible failure in a harness with this gate. Offload is nearly free
  (12→28 CPU-MoE layers costs 6.5% of generate); quant is not.
- **128K buys nothing.** 11–17% slower, byte-identical commits. Every real code
  file in the repo fits in 64K, including a 222 KB Dart file at ~41k tokens
  (Dart tokenises at ~5.4 bytes/token here).
- Generation slows with **KV depth**, not offload: 20 tok/s at 11.7k prompt,
  17.0 at 29.2k, 15.4 at 40.8k. A big prompt is paid twice. Another argument for
  `Files:`.
- The first request after a manual `systemctl restart llama.service` pays a HIP
  JIT compile at ~5–6 tok/s. `llama-warmup.service` pays it at boot only.

---

## 13. Invariants — the rules that have already cost us something

Each of these is a bug we shipped or nearly shipped. They are not style
preferences.

1. **Never `npm ci`, `git checkout`, or commit inside the Z440's repos.** The
   agents share those checkouts.
2. **Look, don't change:** `~/harness`, `~/.paperclip`, `/etc/deepseek-meter`,
   systemd services, anything under `/home/agent`. Change them only when
   explicitly asked, with a backup and an idle board.
3. **Do not un-assign and re-assign a unit to wake Zed.** Units block their
   verify issue, and Paperclip's orphan-blocker recovery hands an unassigned
   blocker to the agent that created it (Glorp) within the same second. Use
   `POST /api/agents/<id>/wakeup {"payload":{"issueId":"<id>"}}`.
4. **Assignment wakeups fire on a *change* of assignee.** Re-assigning to the
   holder wakes nothing. (For a *goal* you created, unassign-then-assign is
   safe.)
5. **Test with a real assignment, never `paperclipai agent wake`.** A manual wake
   creates an `on_demand` run with **no bound issue**, so Paperclip treats every
   issue write as cross-issue and fail-closes with a 403. A dependency wakeup
   binds both `issueId` and `taskId` and is fine. A manual wake exercises a
   weaker path and will mislead you.
6. **Only an explicit base is fast-forwarded** (§7). Everything else is a
   starting point.
7. **A unit must change every file it names, with real content.** A zero-byte
   create used to pass: the gate tested `--name-only`, and an empty spec
   contributes no assertions while the empty module is never required. `jobd`
   now uses `--numstat` per pathspec and fails with exit 6 unless that file's own
   commits add or delete a line. Binary files and deletions still pass.
8. **Code and its tests belong in the same unit.** Split across units, unit 1
   ships a module with two functions missing and goes green because nothing tests
   it yet — and then the next unit is told to fit the tests to the module. Never
   bend tests to code.
9. **Name what must survive, not only what must not break.** A task that said
   "add a JSDoc block, do not change executable code" got twelve lines of design
   rationale deleted. The model narrated the decision; it was Qwen's call, not
   aider's or the harness's. `CONVENTIONS.md` now forbids it and the identical
   re-run went from `2 insertions, 12 deletions` to `4 insertions` — but both
   fixes are prompt-level against a 3B-active model, and this failure class is
   not one tests can catch.
10. **Never paste secrets, credentials or customer data into an issue.** Glorp's
    text goes to DeepSeek's API, a different provider under a different data
    policy.
11. **`read_issue` descriptions by id, never from a list.** Paperclip's list
    endpoint truncates at `ISSUE_LIST_DESCRIPTION_MAX_CHARS` (1200). Zed once
    sent a task clipped mid-file at exactly character 1200; 29 past units had
    descriptions over that, and several "under-applied" results are probably
    this. Both bridges now re-read by id and refuse to dispatch if that fails.
12. **Every exit path must clear the checkout.** A failed `PATCH` once left an
    issue `in_progress`, and the crash-recovery re-claim dispatched it again —
    four wakeups, four real GPU jobs, zero recorded results. `report()` now
    records the result as a comment, resets to `todo`, and releases the checkout
    even if that fails. Crash recovery is right to retry; it is only safe when
    non-crash exits are reliable.
13. **A failing test suite is data, not a crash.** `jobd` deliberately runs
    without `set -e`; every exit path emits valid JSON.
14. **Fix the write, not the symptom.** The worktree leaks were all filter bugs
    downstream of something writing where it shouldn't. `gitignore: false`,
    `.git/info/exclude`, and `jobd` unlinking its own symlinks are the fixes; a
    widened regex was the trap. A filter tuned on a repo with no `.gitignore`
    silently stopped working on one that has it.
15. **`goal-fanout`'s `MAX_UNITS` is the only real cap on fan-out width.**
    `CROSS_ISSUE_INFLUENCE_LIMIT = 20` has **no call site on issue creation** —
    it covers `update`, `comment` and `interaction_resolution` only. Paperclip
    places no bound on how many children a run creates.
16. **Don't change the `charlie` password or add SSH keys.** Ask Charlie. That
    password unlocks passwordless sudo, `ssh z440` as a sudoer, and the DeepSeek
    key.

---

## 14. Diagnostics

All read-only.

```bash
# mini PC health
systemctl --user status paperclipai z440-tunnel
curl -s -o /dev/null -w '%{http_code}\n' localhost:3100/api/health
curl -s localhost:4100/_meter/status | jq '{open, windows}'
journalctl --user -u paperclipai -n 50 -o cat
journalctl -u z440-sorbit-push -n 3 -o cat      # is the Z440's clone fresh?
journalctl -u infra-reader.service              # Glorp's read audit trail
deepseek-usage --balance                        # spend, cache rate, drift

# the board (the CLI defaults to key=value text — pass --json)
paperclipai issue list --company-id ddeaf594-a0a2-4847-bd99-db9b2866cf0d --json
paperclipai issue get <key-or-id> --json        # `list` TRUNCATES descriptions
paperclipai activity issue <id> --json          # NB: rejects --company-id

# Z440
ssh z440 'systemctl is-active llama litellm; curl -s localhost:8080/health; df -h / | tail -1'
ssh z440 'sudo -u agent flock -n /home/agent/.lock true && echo "GPU free"'
ssh z440 'sudo -u agent ls /home/agent/work /home/agent/verify'   # both empty at rest
ssh z440 'sudo -u agent tail -f /home/agent/logs/<id>.log'
ssh z440 'journalctl -u llama.service -f'

# read a goal branch without touching it
B=agent/goal-<slug>
ssh z440 "sudo -u agent git -C /home/agent/repos/sorbit diff --stat prod-testing...$B"
ssh z440 "sudo -u agent git -C /home/agent/repos/sorbit log -p prod-testing..$B" | less
```

Gotchas that will waste your time otherwise:

- Run git on `/home/agent/*` as **`sudo -u agent`**, not bare `sudo` — plain
  `sudo` gets `fatal: detected dubious ownership`.
- Globs in a remote command expand **locally as `charlie`**. Wrap them:
  `ssh z440 'sudo sh -c "ls /home/agent/work/*"'`.
- `issue list` omits `executionState` entirely — it comes back `null`. Fetch
  issues individually if you care about monitor state.
- `flock` is held by the *suite*, not the parent: a hard-killed job leaves the
  lock held by its orphaned children until they exit. That is correct — the lock
  means "a suite is running". The next run queues behind it.
- A forced command runs with `no-pty`, so **sshd sends no SIGHUP** when the
  client goes away. Killing the client does not stop the remote run. `verifyd`
  sweeps on the assumption that any worktree present while it holds the lock
  belongs to a dead run.
- After a restart, expect one `graceful heartbeat run drain failed {signal:
  SIGTERM}` from the **old** process shutting down against an
  already-stopping Postgres. Count errors from `ActiveEnterTimestamp`, not from
  the top of the journal.

---

## 15. Changing the harness safely

1. **Check the board is idle first.** No `aider`/`jobd` processes, `/home/agent/jobs`
   and the worktree dirs empty, no issue `in_progress`, GPU lock free.
2. **Back up in place**, `<file>.bak-<prior-state>`.
3. **Test on `scratch`, or a throwaway repo**, not on Sorbit. Several of these
   fixes were verified against the real commits of the run that exposed them —
   that is the standard: prove the old code fails and the new code passes on
   real data.
4. **Prefer a drop-in to an edit** for anything a managed install regenerates.
   `paperclipai service install` rewrites the unit file; it does not touch
   `*.service.d/`.
5. **Don't patch the managed Paperclip install.** If the fix is upstream,
   document it as upstream and leave it. (Current example: §16.)
6. **Append a change-log entry to `FANOUT.md`'s "Since this was written"** — what
   broke, how it was proved, what was verified, and the backup filename. That
   section is the harness's real history.
7. Remember `glorp-sync-tools` if you touched anything in `/opt/glorp`'s source.

---

## 16. Known open items

| Item | State |
|---|---|
| **Flutter tests don't run** | `flutter analyze` only; 115 `*_test.dart` files idle |
| **Integration / Postgres / live suites** | Cannot run, by design (docker = root on the Z440) |
| **Z440 disk** | 47 GB free of 231 GB (2026-11-16, with the Qwen3.6 trial model on disk; 25 GB on 2026-09-18). `jobd` and `verifyd` have **no** free-space check; only `repo-add` does. A full disk fails jobs mid-run. |
| **Sorbit only** | The one repo onboarded. `Z440_DEFAULT_REPO` should be dropped when a second arrives. |
| **Stale `triggered` monitor** | Every unit that used a retry (SOR-89, SOR-83, SOR-65) ends `done` with `executionState.monitor.status: "triggered"`. **Inert** — the trigger nulled `monitorNextCheckAt` and the dispatch query requires it non-null, so it cannot fire even if a human reopens the unit. Not fixable from the harness: Paperclip rebuilds the state in `derivePersistedMonitorState`, which returns `triggered` whenever `attemptCount > 0`, and overwrites any `executionState` an agent submits. Upstream display/audit behaviour. Left alone. |
| **Stranded units** | A unit can sit `in_progress` with no run (SOR-97, 2026-09-30: its assignment wake was consumed by a run working on SOR-96). Zed only wakes on assignment, so nothing recovers it. Cause is read from run records, not reproduced. `z440-stale-sweeper` handles Zed (never yet fired live); to recover by hand, `POST /api/agents/<zed>/wakeup {"payload":{"issueId":"<id>"}}`. Verity is not swept. |
| **`local_trusted`** | No per-person audit trail; any local process on the mini PC is the board (§10) |
| **LiteLLM master key** | No DB, so no virtual keys; the agent authenticates with the master key. Rotating means `/etc/litellm/env` *and* `/home/agent/.env`. |
| **Test artifacts** | `agent/goal-smoke` + `agent/*` branches on Sorbit and 14 on `scratch`, unmerged so `agent-prune` leaves them. Issue SOR-4 parked `in_review`. Delete by hand when done. |
| **`df` free-space refusal in `repo-add`** | In place, never exercised — testing it means filling a 231 GB disk |

---

## Where to go deeper

| Doc | Read it for |
|---|---|
| [`ENGINEER-GUIDE.md`](ENGINEER-GUIDE.md) | Tailscale + SSH setup, how to write a good goal, what to do with each board state. The doc to hand a new engineer. |
| [`FANOUT.md`](FANOUT.md) | The fan-out build phase by phase, and **"Since this was written"** — the dated change log of every fix, with its proof and backup file. Start here for "why is it like this?" |
| [`MINIPC.md`](MINIPC.md) | The orchestrator as built: Paperclip, the agents' config, the quota guard, and the whole DeepSeek meter / Glorp isolation design with its measurements |
| [`Z440.md`](Z440.md) | The executor as built: model server operation, the tuning benchmarks, aider, `jobd`, worktree-leak archaeology |
| [`README.md`](README.md) | One-screen overview and the health snippet |
| [`history/`](history/) | The original plans. **Not current** — where they disagree with the above, the above wins. |

**Two reading habits.** `FANOUT.md`'s change log is ordered by date, so the last
entries are the current behaviour; earlier text in the same doc may describe what
a later entry replaced. And the `.bak-*` files on both boxes are the ground truth
for when something changed.
