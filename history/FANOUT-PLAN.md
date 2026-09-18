# Fan-out Plan: decompose → edit → test → verify

Build plan for the multi-stage flow: an orchestrator goal is decomposed into
small units, each edited by the local model with a scoped file list, each
unit-tested as it lands, and the whole thing verified against the full suite
before it is called done.

Companion to `HARNESS-PLAN.md` (the original two-box build) and
`../Z440.md` (what is actually deployed today). Read `../Z440.md` first;
this plan assumes that state.

Status: **all six phases built and verified; Phase 5 (loop-back) switched on
2026-09-11 and its red path has since run live.** The flow runs end to end with no human in it: a goal fans out,
units land on an integration branch one at a time, the suite verifies the
branch, and the goal comes back to a person. Phase 3 was built *without* the
plan's accepted-plan gate — see `../FANOUT.md`. See `../FANOUT.md` for what was actually
deployed and where this plan turned out to be wrong — notably that
`npm run test:unit` is *not* hermetic, and that both of the "verify this before
building" questions (Phase 3's adapter gate, Phase 5's run-context 403) resolved
in the build's favour.

The constraints table below is as of the plan (2026-09-08). Two have moved:
the executor now runs a **64K** window, not 32K; and Flutter is installed, so
touched Dart files get `flutter analyze` (not `flutter test`). The planner is on
DeepSeek, not the Claude subscription (`../MINIPC.md`).

Phases are ordered by dependency. Phases 1 and 2 are worth doing on their own
merit even if the rest is never built.

---

## The constraints that shape this design

These are measured, not assumed. Every design decision below falls out of one
of them.

| Constraint | Consequence |
|---|---|
| One GPU. `llama-server --parallel 1`, `jobd` holds an `flock` for the whole job | **Fan-out is not parallel.** N units queue. Decomposition buys prompt quality, not wall-clock. |
| 32K context, `edit-format: diff` | A unit must fit one file's worth of context. ~318 of Sorbit's 338 Dart files qualify; the other 20 need refactoring, not hardware. |
| `agent` has no sudo and is not in `docker` | Integration tests that need `db:start` (docker postgres) **cannot run.** Unit tests can. |
| Claude Pro, 50% weekly self-imposed cap | One planning call per goal, not one subagent per unit. The orchestrator is the metered resource; the Z440 is the free one. |
| `jobd` branches every job from the repo's checked-out HEAD | Units produce N sibling branches that never combine. **Nothing to verify.** Phase 2 fixes this. |

The serialization is not only a cost. Because units run strictly one at a time,
fan-in is trivial: each unit can branch from the integration tip and
fast-forward it back, with no merge conflicts by construction. Phase 2 relies
on this.

---

## What already exists — do not rebuild

| Capability | Where |
|---|---|
| Plan → child issues | Paperclip native: `POST /api/issues/:id/accepted-plan-decompositions`, backed by `issue_plan_decompositions` (`sourceIssueId`, `acceptedPlanRevisionId`, `childIssueIds`). Requires an accepted plan *document revision*. |
| "Wait for all children" | `issue_relations` (`type: "blocks"`) + `issue-dependency-wakeups`. A verify issue blocked by all children wakes when the last one closes. |
| Pause/cancel a whole tree | `issue-tree-control.ts`, `issue_tree_holds` / `issue_tree_hold_members`. |
| Scoped file context | `Files:` field → `jobd` `.files` → aider argv. Measured 3.2k tokens on a real Sorbit edit. |
| Extra prose/snippet context | Already works. The issue body is free-form and is passed to `aider --message`. |
| Per-unit test execution | `jobd` `.tests` field: runs a shell command, returns its exit code, and it outranks nothing (aider failure outranks it). **The mechanism exists; there is no toolchain to run.** |
| Retry and escalation | `z440-exec` 3-strike rule → `blocked` for human review. |

The only genuinely absent pieces are: a toolchain, an integration branch, a
verify agent, and the loop-back.

---

## Phase 1 — Node 20, and per-unit unit tests

**Why first:** without this, `exit 0` means "aider finished", not "the code
works". Every later phase is verification machinery with nothing to verify.
This is the single highest-value change on the list.

### 1.1 Install Node 20

`cloud-run/package.json` declares `"engines": {"node": "20"}`. Ubuntu's apt
candidate is 18.19.1 — **not sufficient.** Use NodeSource.

```bash
# on the Z440, as jack (passwordless sudo)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
node -v          # expect v20.x
```

`agent` needs no special grant — `/usr/bin/node` is world-executable. Confirm:

```bash
sudo -u agent bash -lc 'node -v && npm -v'
```

> **Do not** add `agent` to the `docker` group to make integration tests run.
> That group is root-equivalent and would undo the account's isolation, which
> is the whole reason the Sorbit clone exists separately from `/home/jack`.
> Integration tests stay a human step. This is a deliberate ceiling.

### 1.2 Install dependencies once, in the main clone

`package-lock.json` is present, so `npm ci` is reproducible. Registry is
reachable from the Z440 (verified, HTTP 200).

```bash
sudo -u agent bash -lc '
  cd /home/agent/repos/sorbit/app-frontend/sorbit_app/cloud-run && npm ci'
```

Worktrees do not inherit `node_modules` (it is untracked, and `jobd`'s JUNK
filter already ignores it). Two options:

- **Chosen:** symlink from the worktree at job setup. Add to `jobd`, after the
  worktree is created:
  ```bash
  # node_modules is untracked and large; share the main clone's install
  # rather than reinstalling per job.
  for nm in $(cd "$REPO" && find . -name node_modules -maxdepth 4 -type d 2>/dev/null); do
    [ -e "$WT/$nm" ] || ln -s "$REPO/${nm#./}" "$WT/$nm"
  done
  ```
- Rejected: `npm ci` per job. ~40s and a registry round-trip per unit, for an
  identical tree.

A symlinked `node_modules` is shared mutable state across jobs. It is
install-only (tests do not write to it), and jobs are serialized, so this is
safe — but it means a `npm ci` must not run while a job holds the lock.

### 1.3 Wire the per-unit test command

No code change needed — `jobd` already honours `.tests`, and `z440-exec`
already parses a `Tests:` field. The unit-test target is:

```
Tests: cd app-frontend/sorbit_app/cloud-run && npm run test:unit
```

46 unit test files, mocha 11, hermetic (no postgres, no docker). This is the
only Node suite that can run under the constraints; `test:integration*`,
`test:postgres*` and `test:live*` all need a database.

### 1.4 Flutter — defer, decide later

`app-frontend/sorbit_app/test/` holds **115** `*_test.dart` files across
`widget/` and `unit/`. `flutter test` would cover them. The Flutter SDK is a
~1 GB install plus a pub cache. The Z440 has **44 GB free of 231 GB (81% used)**
— headroom is real but not generous once per-repo `node_modules` trees and
pub caches accumulate. Check before committing:

```bash
df -h /home/agent
```

Defer until Phase 1.3 is proven on the Node side. If the app code (not
`cloud-run`) is where the work actually is, this stops being optional.

### Acceptance

- [ ] `sudo -u agent node -v` → v20.x
- [ ] `npm run test:unit` passes in the main clone, run as `agent`
- [ ] A job with a `Tests:` line returns `exit 0` on passing tests
- [ ] A job with a deliberately broken edit returns **non-zero**, and
      `z440-exec` records a strike — this is the test that matters, because it
      proves the signal is real rather than always-green
- [ ] `/home/agent/work` is empty afterwards (no worktree leak from the
      symlinks — `node_modules/` is in the JUNK filter, but a *symlink* is not
      a directory; verify `git status --porcelain` is still clean)

> The symlink/JUNK-filter interaction is the likely bug in this phase. The
> existing filter matches `node_modules/` as an untracked directory. A symlink
> shows up as `?? node_modules` with no trailing slash and will **not** match.
> Expect to widen the regex to `node_modules/?` and re-verify cleanup.

---

## Phase 2 — an integration branch

**Why:** today every job does `worktree add -b agent/<id> <WT> $BASE` where
`$BASE` is the repo's checked-out HEAD. Ten units produce ten sibling branches
all rooted at `prod-testing`, never combined. There is no branch on which a
full suite could be run, so Phase 4 is impossible without this.

### 2.1 Accept a base in the job contract

`jobd` contract in becomes `{"repo","task","files"?,"tests"?,"base"?}`.

```bash
base=$(jq -r '.base // empty' <<<"$JOB")
if [ -n "$base" ]; then
  case "$base" in *' '*|-*) die "invalid base: $base" ;; esac
  git -C "$REPO" rev-parse --verify "refs/heads/$base" >/dev/null 2>&1 \
    || die "no such base branch: $base"
  BASE="$base"
else
  BASE=$(git -C "$REPO" symbolic-ref --short HEAD 2>/dev/null) \
    || BASE=$(git -C "$REPO" rev-parse HEAD)
fi
```

### 2.2 Fast-forward the base on success

Because the GPU lock serializes jobs, the base cannot move while a job holds
it. So fan-in is a fast-forward, not a merge — and if it is *not* a
fast-forward, something moved underneath us and we must not guess.

Add after the test step, still under the lock, only when `rc -eq 0` and a
`base` was supplied:

```bash
if [ "$rc" -eq 0 ] && [ -n "$base" ]; then
  if git -C "$REPO" merge-base --is-ancestor "refs/heads/$base" "refs/heads/$BR"; then
    git -C "$REPO" update-ref "refs/heads/$base" "refs/heads/$BR" \
      && echo "[jobd] fast-forwarded $base to $BR" >>"$LOG"
  else
    echo "[jobd] $base moved underneath this job; NOT advancing it" >>"$LOG"
    rc=75   # EX_TEMPFAIL - the edit is fine, the integration is not
  fi
fi
```

Never advance the base on a failing job. A failed unit leaves its `agent/*`
branch for inspection and the integration branch untouched.

### 2.3 Create the integration branch per goal

The orchestrator creates it once before dispatching units. Needs a way to run
a plain git command on the Z440, which the forced-command key deliberately
forbids. Options:

- **Chosen:** a `create-base` verb in the job contract — `jobd` grows
  `{"op":"create-base","repo":...,"base":...,"from":?}` handled before the
  lock, creating the branch if absent and returning immediately. Keeps the
  attack surface at exactly one forced command.
- Rejected: a second SSH key with a git-only forced command. More keys, more
  surface, no benefit.

### 2.4 Reflect the branch in `z440-exec`

Add a `Base:` field alongside `Repo:`/`Files:`/`Tests:`/`Branch:`, and extend
the strip regex — currently `^\s*(Repo|Tests|Branch|Files)\s*:.*$` — to include
it. **This regex has been forgotten once already** (the `Files:` line leaked
into the task text sent to the model). Change both together.

### Acceptance

- [ ] Two sequential jobs with the same `base` produce an integration branch
      carrying **both** commits
- [ ] A failing job leaves the base at the previous commit
- [ ] A base moved by hand mid-flight produces exit 75 and no ref update
- [ ] Omitting `base` reproduces today's behaviour exactly (regression guard)

---

## Phase 3 — decomposition without a subagent per unit

**Why not subagents:** each spawn re-derives context cold and costs metered
Claude quota to produce a prompt for a model that will execute it serially
anyway. One planning call produces the same N units.

### 3.1 Glorp emits a structured plan

Glorp (`claude_local`) writes a plan document listing N units, each carrying
the fields `z440-exec` already parses:

```
## Unit 3 — logging module returns structured errors
Repo:  sorbit
Base:  agent/goal-<slug>
Files: app-frontend/sorbit_app/cloud-run/src/modules/shared/logging.js
Tests: cd app-frontend/sorbit_app/cloud-run && npm run test:unit
<free-form task text, including any snippet context worth pasting inline>
```

Unit sizing rule: **one unit edits one file.** Aider sends whole files, so two
files means two whole files in a 32K window. The `Files:` list may name more
than one, but that is the exception and needs a token budget check.

### 3.2 Accept the plan, then decompose

`POST /api/issues/:id/accepted-plan-decompositions` requires an
`acceptedPlanRevisionId` — a plan document revision that has been *accepted*,
not merely written. That gate is a feature: a human sees the decomposition
before N GPU-hours are spent on it.

Confirm before building: the exact request body accepted by
`createAcceptedPlanDecompositionSchema`, and whether a `process` agent (Glorp
is `claude_local`; Zed is `process`) may call the route at all — the
`decomposeAcceptedPlan` path takes `ownerAgentId`/`ownerUserId`/`ownerRunId`
and runs under the same run-context rules that produced the earlier 403.
`server/src/routes/issues.ts:9546` and `:9630` are the code to read.

`requestFingerprint` plus the `(companyId, sourceIssueId, acceptedPlanRevisionId)`
unique index make the call idempotent — a retried decomposition will not
double-create children, but a *changed* child set against the same revision
throws `conflict`. Do not mutate a plan and re-post it under the same revision.

### 3.3 Dependency edges

For each child, add `issue_relations` rows so the verify issue (Phase 4) is
blocked by all of them.

### Acceptance

- [ ] An accepted plan with 3 units produces 3 child issues with correct
      `Repo:`/`Base:`/`Files:`/`Tests:` fields
- [ ] Re-posting the same revision is a no-op, not 3 more children
- [ ] Zed drains all 3 serially with no operator intervention
- [ ] The integration branch carries all 3 commits

---

## Phase 4 — the verify agent

**Why it is separate:** the full suite needs no GPU. If it ran through `jobd`
it would take the GPU lock and block model work for nothing.

### 4.1 A second forced command

New script `/home/agent/bin/verifyd`, new SSH key, forced command as before.
Contract: `{"repo","base","suite"?}` → `{"exit","log"}`. It does **not** take
`/home/agent/.lock`.

It creates a detached worktree at the integration branch tip, symlinks
`node_modules`, runs the suite, tears the worktree down, returns the exit code.

Run it niced. `llama-server` uses 6 of 12 cores with `--n-cpu-moe 12`, so a
concurrent test run has headroom but will slow generation:

```bash
nice -n 10 timeout "$SUITE_TIMEOUT" bash -c "$suite"
```

### 4.2 A `verify` Paperclip agent

A second `process` agent pointed at a `z440-verify` bridge script — the same
shape as `z440-exec` but calling `verifyd`. It is woken by the dependency
wakeup when the last child closes.

Green → move the goal issue to `in_review`, assigned to a human (the
`invalid_issue_disposition` 422 lesson: an agent may not move an issue to
`in_review` without naming who owns the next action).

Red → Phase 5.

### 4.3 Suite scope

Full suite here means `npm run test:unit` across the repo — **not** `npm test`,
which globs `test/**/*.test.js` and pulls in integration tests that need
postgres. If Flutter lands in Phase 1.4, add `flutter test`.

So "full suite" is narrower than it sounds, and the gate is correspondingly
weaker. Say so on the issue when reporting green, or the label will be trusted
for more than it covers.

### Acceptance

- [ ] Verify runs while a GPU job is in flight, and neither blocks the other
- [ ] Green sets the goal issue to `in_review` with a human assignee
- [ ] Red leaves it open with the failure output attached
- [ ] Killing the verify mid-run leaves no worktree behind

---

## Phase 5 — loop-back, with a hard cap

Red suite → parse the mocha output into failing files → create child issues
scoped to those files → they drain through the same path → verify re-runs.

### 5.1 The cap is the whole design

An unbounded fix-retest loop on a 3B-active model will burn GPU hours
converging on nothing, and each iteration re-edits files a previous iteration
already touched.

- **Max 2 loop-back rounds per goal.** Then `blocked`, human review.
- A file that has failed in **two** rounds is excluded from a third — the model
  is not going to get it on attempt three, and the existing 3-strike rule
  operates per-issue, not per-file, so it will not catch this.
- Record the round number on the goal issue so the cap survives a restart.

### 5.2 Attribution

Child issues created by the verify agent are cross-issue writes. They need a
bound run — `PAPERCLIP_TASK_ID` is set for *assignment* wakeups but absent for
`agent wake`, and an unbound run is refused with
`cross_issue_influence_run_context_required`. The dependency wakeup must bind
the goal issue, or every loop-back write 403s. **Verify this before building
5.1**; it is the same failure that cost four GPU jobs during the initial build.

Also relevant: `CROSS_ISSUE_INFLUENCE_LIMIT = 20` per run. A round creating
more than 20 children will be capped.

### Acceptance

- [ ] A deliberately broken unit produces a scoped fix issue naming the right file
- [ ] Round 2 failure escalates to `blocked` rather than starting round 3
- [ ] A file failing twice is not re-dispatched
- [ ] No 403s on the loop-back writes

---

## Phase 6 — onboarding a new repo

**Why:** Sorbit took an afternoon of ad-hoc decisions — where to clone from,
which branch is the base, which test target is hermetic, which files fit the
context window, and one genuine near-miss on a live credential. None of that
was written down as a procedure. The second repo should take fifteen minutes,
and the credential check must not depend on anyone remembering it.

This phase is independent of Phases 1–5 and can be built at any point, but the
manifest in 6.1 is what lets Phase 3 stop repeating `Tests:` on every issue.

### 6.1 A per-repo manifest

Today every issue must carry `Repo:`, `Files:`, `Tests:` and (after Phase 2)
`Base:`. Three of those four are properties of the *repo*, not the task, so
they get repeated on every issue and drift.

Move them into a manifest, kept **outside the working tree** so aider can
never edit it and it is never committed to the target repo:

`/home/agent/etc/repos/<name>.json`

```json
{
  "name": "sorbit",
  "base": "prod-testing",
  "tests": "cd app-frontend/sorbit_app/cloud-run && npm run test:unit",
  "setup": "cd app-frontend/sorbit_app/cloud-run && npm ci",
  "node_modules": ["app-frontend/sorbit_app/cloud-run"],
  "max_file_bytes": 70000,
  "notes": "Integration/postgres suites need docker; not available to agent."
}
```

`jobd` resolves each field as **job value → manifest value → built-in default**,
so an issue may still override anything, but does not have to say anything. Add
near the top of `jobd`, after `$REPO` is validated:

```bash
MAN="/home/agent/etc/repos/$repo.json"
mfield() {  # mfield <json-key> <job-value>
  [ -n "$2" ] && { printf '%s' "$2"; return; }
  [ -f "$MAN" ] && jq -r --arg k "$1" '.[$k] // empty' "$MAN" 2>/dev/null
}
tests=$(mfield tests "$tests")
base=$(mfield base "$base")
```

A missing manifest is not an error — it degrades to exactly today's behaviour.

### 6.2 `repo-add`

`/home/agent/bin/repo-add`, run on the Z440. Does the mechanical work and
**stops** at the two things a script must not decide alone (secrets, and which
test target is hermetic).

```bash
#!/usr/bin/env bash
# repo-add <name> <source>   source = a git URL or a local path
# Clones, surveys, and writes a manifest stub. Does NOT decide that the repo
# is safe to expose to the model - see the secret gate it prints.
set -euo pipefail
name=$1; src=$2
case "$name" in */*|.*|"") echo "invalid repo name" >&2; exit 2 ;; esac
dest="/home/agent/repos/$name"
[ -e "$dest" ] && { echo "$dest already exists" >&2; exit 2; }

avail=$(df --output=avail -BG /home/agent | tail -1 | tr -dc 0-9)
[ "$avail" -ge 10 ] || { echo "only ${avail}G free; refusing" >&2; exit 2; }

# Clone, never copy. A clone takes tracked content only, which is what left
# Sorbit's untracked Firebase admin key behind on the source machine.
git clone "$src" "$dest"
cd "$dest"
base=$(git symbolic-ref --short HEAD)

echo "=== size survey (files the 32K window can hold) ==="
git ls-files -z | xargs -0 -r stat -c '%s %n' 2>/dev/null \
  | sort -rn | awk '
    {n++; if($1<=20000)a++; else if($1<=70000)b++; else c++}
    END{printf "  <=20KB: %d   20-70KB: %d   >70KB (too big to edit): %d\n",a,b,c}'
echo "  largest tracked files:"
git ls-files -z | xargs -0 -r stat -c '%s %n' 2>/dev/null | sort -rn | head -5 \
  | awk '{printf "    %8.1f KB  %s\n", $1/1024, $2}'

echo "=== toolchain hints ==="
git ls-files | grep -E '(^|/)(package\.json|pubspec\.yaml|pyproject\.toml|go\.mod|Cargo\.toml)$' | head -10

mkdir -p /home/agent/etc/repos
cat > "/home/agent/etc/repos/$name.json" <<JSON
{
  "name": "$name",
  "base": "$base",
  "tests": "",
  "setup": "",
  "node_modules": [],
  "max_file_bytes": 70000,
  "notes": "UNREVIEWED - secret gate not yet signed off"
}
JSON

cat <<'GATE'

  ---------------------------------------------------------------
  NOT READY YET. Two things a script must not decide:

  1. SECRET GATE. Run the checks in FANOUT-PLAN.md 6.3 and clear
     the "UNREVIEWED" note by hand. A clone excludes *untracked*
     secrets; it does NOT exclude committed ones, and it brings
     the full history with it.

  2. TEST TARGET. Pick a suite that needs no docker, no network
     and no live credentials, and put it in "tests".
  ---------------------------------------------------------------
GATE
```

### 6.3 The secret gate

This is the step that matters, and it is the reason `repo-add` refuses to mark
a repo ready on its own.

Sorbit's working tree held `sorbit-1a42f-firebase-adminsdk-*.json` — a live
Firebase **admin** credential. Cloning rather than sharing the directory left
it behind, and that was verified. But that outcome depended on the file being
*untracked*. A committed credential would have come across, and `git clone`
brings the whole history, so a secret that was committed and later deleted is
still in the pack.

Run all three, on the clone:

```bash
cd /home/agent/repos/<name>

# 1. credential-shaped filenames, anywhere in history
git log --all --diff-filter=A --name-only --pretty=format: \
  | sort -u | grep -Ei '(secret|credential|adminsdk|serviceaccount|\.pem$|\.p12$|\.pfx$|\.key$|id_rsa|\.env($|\.))' 

# 2. high-signal content in the current tree
git grep -nIE 'BEGIN [A-Z ]*PRIVATE KEY|"private_key"|AKIA[0-9A-Z]{16}|ghp_[A-Za-z0-9]{36}|sk-[A-Za-z0-9]{20,}' -- . 

# 3. what .gitignore is protecting - each entry is a hint about what kind of
#    secret the repo expects to exist on a developer machine
grep -vE '^\s*(#|$)' .gitignore 2>/dev/null | head -40
```

Better: install `gitleaks` (a single static binary, no runtime) and run
`gitleaks detect --no-git=false`. It is not installed on the Z440 today.

Treat all of this as a **net, not a guarantee**. The sign-off is a person
deciding the repo is safe to expose to a model that will read any file it is
pointed at, and whose output a human will later merge. Record the decision in
the manifest `notes`.

> If a repo *does* contain a committed live credential, onboarding is not the
> problem to solve — rotate the credential first. Do not onboard around it.

### 6.4 Toolchain and dependencies

Fill in `setup` and `node_modules` in the manifest, then run setup once as
`agent` in the main clone (never inside a worktree — the install is shared, per
Phase 1.2).

Per-language notes:

| Ecosystem | Setup | Hermetic test target |
|---|---|---|
| Node | `npm ci` | a `test:unit`-style script; avoid anything invoking docker or a live DB |
| Python | `python3 -m venv .venv && .venv/bin/pip install -e '.[test]'` | `pytest -m "not integration"` |
| Dart/Flutter | `flutter pub get` | `flutter test` |
| Go | none | `go test ./...` |
| Rust | none | `cargo test` |

The rule for choosing the target: **no docker, no network, no live
credentials.** `agent` has none of the three and must not be given them.

### 6.5 Context-fit survey

`repo-add` prints the size buckets. Read them before promising the harness can
work on the repo:

- **`>70 KB` count near zero** — the repo is a good fit; the model can be
  pointed at most files.
- **A large `>70 KB` bucket** — expect to name files narrowly and accept that
  some are simply out of reach. Sorbit: 20 of 338 Dart files.

This is a property of the repo, not a setting to tune. A repo full of 200 KB
files is not a context-window problem to buy hardware for.

### 6.6 Register with Paperclip

- Set `Z440_DEFAULT_REPO` in the dispatching agent's `adapterConfig.env` **only
  if that agent works one repo**. For multiple repos, drop the default and make
  `Repo:` mandatory on issues — a wrong default silently edits the wrong repo.
- One dispatching agent per repo is cleaner than one agent switching repos:
  the checkout, strike count and reviewer assignment are all per-agent.

### 6.7 The fifteen-minute checklist

```
[ ] repo-add <name> <source>          — clone, survey, manifest stub
[ ] secret gate (6.3), all three checks, decision recorded in notes
[ ] pick the hermetic test target, put it in manifest "tests"
[ ] run manifest "setup" once as agent in the main clone
[ ] clear "UNREVIEWED" from manifest notes
[ ] smoke: one trivial issue (add a comment to one small file) end to end
[ ] confirm /home/agent/work is empty afterwards
```

### Acceptance

- [ ] `repo-add` on a fresh repo produces a working clone and a manifest stub
- [ ] It refuses when free space is under 10 GB
- [ ] It refuses a name containing `/` or a leading `.`
- [ ] A job naming only `Repo:` and `Files:` inherits `base` and `tests` from
      the manifest
- [ ] An issue-level `Tests:` still overrides the manifest
- [ ] A repo with no manifest behaves exactly as today (regression guard)
- [ ] The smoke issue reaches a reviewable branch with no operator intervention

---

## Out of scope

- **Parallel model execution.** One GPU, one KV cache. Not a software problem.
- **Integration / postgres / live test suites.** They need docker, which needs
  a group grant that would make `agent` root-equivalent.
- **Pushing to GitHub.** The clone's origin is set but has no key. Deliberate:
  the model's output reaches the remote only by a human hand.
- **Editing the 20 Dart files over 70 KB.** A 222 KB widget file is a refactor,
  not a context-window purchase.

## Risks

| Risk | Mitigation |
|---|---|
| Symlinked `node_modules` defeats the worktree-leak filter | Explicit acceptance check in 1.3; expect to widen the regex |
| Shared `node_modules` mutated by a job | Tests are read-only against it; jobs serialized; never `npm ci` under the lock |
| Decomposition route rejects a `process` agent | Read `routes/issues.ts:9546` before building Phase 3; fall back to the orchestrator creating children directly via `POST /api/issues` |
| Loop-back 403s on run context | Verified in 5.2 before 5.1 is built |
| Full-suite gate reads stronger than it is | State the actual scope in the green comment |
| A new repo carries a **committed** credential into the clone | Secret gate 6.3 is a hard step in `repo-add`'s output, and the manifest stays `UNREVIEWED` until signed off. Rotate, do not onboard around it |
| `Z440_DEFAULT_REPO` silently sends work to the wrong repo once there are two | Drop the default when an agent serves more than one repo; make `Repo:` mandatory (6.6) |
| Disk: 44 GB free of 231 GB, +Flutter SDK ~1 GB + pub cache + per-repo deps | `df -h` gate in 1.4 and in `repo-add` (Phase 6) |

## Build order summary

1. **Node 20 + per-unit tests** — turns `exit 0` into a real signal. Do this regardless.
2. **Integration branch** — makes any end-of-run verification possible. Do this regardless.
3. **Decomposition** — one planning call, N children.
4. **Verify agent** — cheap final gate, no GPU.
5. **Loop-back** — capped at 2 rounds.
6. **`repo-add` + per-repo manifest** — independent of 1–5; build it as soon as
   a second repo is on the horizon. The manifest also removes the repeated
   `Tests:`/`Base:` fields that Phase 3 would otherwise emit on every issue.
