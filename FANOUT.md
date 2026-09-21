# Fan-out — as built

Companion to `history/FANOUT-PLAN.md`, in the same relationship `Z440.md` has to
`history/HARNESS-PLAN.md`: what is actually deployed, and where the build departed from
the plan. Written 2026-09-08; status re-checked 2026-09-18 (see
[Since this was written](#since-this-was-written)).

**Built: Phases 1–6.** Phase 5 (loop-back) was switched on 2026-09-11. The
two blocking questions the plan asked are both answered in the last section.

## Status by phase

| Phase | State | Notes |
|---|---|---|
| 1 — Node 20 + per-unit tests | **built, verified** | Node 20.20.2; suite is hermetic only with `SKIP_TESTCONTAINERS=1` |
| 2 — integration branch | **built, verified** | `base`, fast-forward fan-in, `create-base` verb |
| 3 — decomposition | **built, verified** | no acceptance gate; `goal-fanout` + Glorp instructions |
| 4 — verify agent | **built, verified** | `verifyd` + `z440-verify` + Paperclip agent **Verity** |
| 5 — loop-back | **built, ON since 2026-09-11** | caps 2 rounds × 4 fix issues; red path exercised live since (SOR-39, 40, 42, 45) |
| **autonomy** | **built, verified** | goal -> units -> verify -> human, 2.5 min, zero touches |
| 6 — repo onboarding | **built, verified** | manifest + `repo-add` + automated secret nets |

## What is new on the Z440

| Path | Purpose |
|---|---|
| `/usr/bin/node` | 20.20.2 from NodeSource (Ubuntu's 18.19.1 candidate is too old) |
| `/home/agent/bin/jobd` | rewritten: manifest, `base`, fan-in, `create-base`, `node_modules` links |
| `/home/agent/bin/verifyd` | **new** — runs a suite against an integration branch, takes no GPU lock |
| `/home/agent/bin/repo-add` | **new** — clone, survey, secret-net, manifest stub |
| `/home/agent/etc/repos/*.json` | **new** — per-repo manifest, outside every working tree |
| `/home/agent/verify/` | **new** — verify worktrees, swept by the next verify |
| `jobd.bak-prefanout` | the previous `jobd`, kept |

On the mini PC: `~/harness/bin/z440-verify` (new bridge), `z440-exec` (gained
`Base:`), and an `ssh z440-verify` host on a second forced-command key
(`~/.ssh/z440_verify`). Paperclip gained a third agent, **Verity**
(`dd57e84b-a6c4-4cd7-9a91-3acfd818f05c`), a `process` agent pointed at the
verify bridge.

> The Z440's address changed to **192.168.1.199** (now static). The old `.120`
> had been reassigned to another host, so the SSH host key check failed loudly
> and `z440-tunnel.service` was running but connecting to nothing. Config,
> tunnel unit and docs updated; the stale `known_hosts` entry removed.

---

## Phase 1 — the correction that mattered

**`npm run test:unit` is not hermetic.** The plan states it is ("46 unit test
files, mocha 11, hermetic (no postgres, no docker)"). It is not: two files under
`test/unit/` — `baseline-computer.test.js` and `expert-qualifications.test.js` —
boot a Postgres testcontainer. As `agent`, with no docker, they fail:

```
Error: Could not find a working container runtime strategy
```

507 passing, 2 failing, **exit 1, permanently**. Had this shipped, every job
would have returned non-zero regardless of the edit — which is exactly as
useless as the always-green signal Phase 1 exists to replace, and harder to
notice, because a red suite looks like the harness working.

The repo already has the right escape hatch. `test/helpers/test-container.js`
documents `SKIP_TESTCONTAINERS=1`, which raises `TestcontainersUnavailableError`
so the suites call `this.skip()`. It did not fire on its own because the helper
only converts a docker error when the message matches
`/docker|ENOENT|ECONNREFUSED|EACCES/i`, and testcontainers' actual message —
"Could not find a working container runtime strategy" — contains none of those
words. So the real test target is:

```
cd app-frontend/sorbit_app/cloud-run && SKIP_TESTCONTAINERS=1 npm run test:unit
```

→ **507 passing, 9 pending, exit 0**, in about 2 seconds.

Fixed in the manifest rather than in the repo: the repo is the model's target,
and the harness should not be committing to it to make itself work.

### Acceptance

- [x] `sudo -u agent node -v` → v20.20.2
- [x] `npm run test:unit` passes in the main clone as `agent` (with the env var)
- [x] A job with a `Tests:` line returns `exit 0` on passing tests
- [x] A job with a failing suite returns **non-zero** — verified with an explicit
      red suite; the base was not advanced and the branch was kept
- [x] `/home/agent/work` empty afterwards, symlinks and all

### The symlink/JUNK-filter bug was real, and bigger than predicted

The plan expected the untracked-symlink form (`?? node_modules`, no trailing
slash) to slip past the `node_modules/` pattern. True — but the anchored filter
`^\?\? ($JUNK)` was *already* broken for anything nested, because Sorbit's
`node_modules` lives at `app-frontend/sorbit_app/cloud-run/node_modules`, not at
the repo root. It had simply never been exercised there.

Fixed three ways rather than by widening one regex:

1. `jobd` **removes the symlinks it created** before the cleanliness check.
   Unlinking a symlink never touches the tree it points at, and it beats relying
   on how `git worktree remove --force` treats a symlink standing where a
   directory belongs.
2. `node_modules` is appended to `.git/info/exclude` alongside `.aider*`, so git
   never reports it — the same shared-common-dir trick, and the same reasoning
   as the `.gitignore` fix: stop the write rather than filter the symptom.
3. The filter gained a `(.*/)?` prefix so junk matches at any depth.

Verified: `/home/agent/work` is empty after every job in this build-out, and
`git worktree list` shows no stale registrations.

---

## Phase 2 — integration branch

Built as specified, with four departures.

**1. A manifest base is a starting point; only an explicit base is advanced.**
The plan's 2.2 fast-forwards "when a `base` was supplied". With the Phase 6
manifest in play, `base` is resolved for *every* Sorbit job (`prod-testing`),
so that reading would fast-forward the repo's trunk to unreviewed model output
on every green job. `jobd` therefore keeps two values apart: `BASE`, the
resolved start point (job → manifest → HEAD), and `base_job`, the explicitly
requested integration branch — and only the latter is ever fast-forwarded.

**2. It refuses to advance the checked-out branch.** `update-ref` on the branch
the main clone has checked out leaves that clone's index describing a tree it no
longer has — it looks like the working copy spontaneously reverted. Caught
before the GPU lock:

```
refusing to use the checked-out branch (prod-testing) as an integration base
```

**3. The diff is taken against a recorded commit, not a branch name.** The
fan-in moves the base *name*, so the plan's `git diff "$BASE...HEAD"` would
report an empty diff for exactly the jobs that succeeded. `jobd` records
`START=$(git rev-parse "$BASE")` at worktree creation and diffs against that.

**4. `create-base` must not resolve its own base.** The generic resolution
requires the base branch to exist; for `create-base` that branch is precisely
the one being created. First cut failed with `no such base branch in sorbit:
agent/goal-smoke`. Now `create-base` resolves its default `from` from the
manifest/HEAD and ignores the job's `.base` as a start point.

Branch names reaching git go through `git check-ref-format` plus an explicit
reject of leading `-` and whitespace, so `--upload-pack=evil` is a name error,
not an argument.

### Acceptance

- [x] Two sequential jobs with the same `base` produce an integration branch
      carrying **both** commits — `agent/goal-smoke` holds `42772ac4` then
      `d7e2c47b`, while `prod-testing` stayed at `fe505ada`
- [x] A failing job leaves the base at the previous commit
- [x] A base moved by hand mid-flight produces **exit 75** and no ref update
      (`agent/goal-smoke moved underneath this job; NOT advancing it`)
- [x] Omitting `base` on a repo with no manifest reproduces today's behaviour

---

## Phase 4 — the verify agent

`/home/agent/bin/verifyd`, second forced-command key, contract
`{"repo","base","suite"?}` → `{"id","repo","base","sha","suite_cwd","exit","log"}`.
Detached worktree at the integration tip, shared `node_modules`, suite run
`nice -n 10` under a timeout, worktree torn down on every exit path.

**A verify green run takes about 3 seconds** and does not touch the GPU lock.
Measured concurrently with a live aider job: verify started 13 s into the job,
returned in 4 s, and the job carried on and finished normally.

### The orphan problem, and why the sweep exists

A forced command runs with `no-pty`. There is no controlling terminal, so when
the SSH client goes away **sshd sends no SIGHUP** — it closes the pipes, and a
suite that is not writing to stdout never notices. Killing the client therefore
leaves the run going. Confirmed directly: after `kill` on the client, the remote
`verifyd`, its `timeout`, and its `sleep` were all still running.

That run cleans up after itself when it ends, so it is not a leak. The real leak
is a run killed outright, which cannot run its trap. So `verifyd` sweeps: it
holds an flock for its whole run, and only one verify runs at a time, therefore
**any worktree still present once we hold the lock belongs to a dead run**.
Verified by `kill -9`, then a normal verify, which swept it and returned green.

One related finding worth keeping: the lock fd is inherited by the suite's
children, so a hard-killed parent leaves the *lock* held by its orphaned
children until they exit. That is not a bug to fix — the lock means "a suite is
running", not "a parent process is alive", and the next verify correctly queues
behind it. The same is true of `jobd` and the GPU lock, where it matters more.

### Honest scope on the green label

Per plan 4.3, every report — green or red — carries:

> Scope: hermetic unit suite only. Integration, postgres and live suites need
> docker, which the `agent` account deliberately does not have, so they did
> **not** run. A green result here is not a full-suite pass.

### Acceptance

- [x] Verify runs while a GPU job is in flight; neither blocks the other
- [x] Green sets the goal issue to `in_review` with a human assignee
      (`assigneeAgentId: null` + `assigneeUserId` in one write, per the
      one-assignee rule)
- [x] Red leaves it open with the failure output attached (exit 4 recorded)
- [x] Killing the verify mid-run leaves no worktree behind — via the sweep

End to end: issue assigned to Verity at 13:29:38, `in_review` with the suite
output attached by 13:29:55. A red-suite issue stayed `todo` with its failure.

---

## Phase 5 — built, switched on 2026-09-11

`z440-verify` grows a loop-back: red suite → failing test files → one scoped fix
issue per file, assigned to Zed, with the verify issue moved to `blocked` on
them. The dependency wakeup then re-runs verify when the last one closes — that
edge is what makes it a loop rather than a fan-out.

**It shipped off (`Z440_LOOPBACK=0`) and was switched on 2026-09-11.** Enabling it lets a red suite create issues
that dispatch real GPU jobs with nobody in the loop. That is a decision to take
deliberately, not something that should start happening because code landed.

Caps, all unit-tested against a mocked API:

- 2 rounds per goal, then `blocked`. Round count is read back from a machine-shaped
  `z440-loopback-round: <files>` marker line in the issue comments, so it
  survives a restart — the run itself holds no state.
- A file that has been dispatched in 2 rounds is excluded from a third and named
  in the comment. The existing 3-strike rule is per-issue and never sees this.
- At most 8 fix issues per round by default, set to **4** on Verity for now, well under the server's
  `CROSS_ISSUE_INFLUENCE_LIMIT` of 20 writes per run.
- No parseable test file in the output → left `todo` for a human rather than
  guessing.

**Failing-file extraction is a heuristic and is documented as one.** Mocha's
spec reporter ties a failure to a file only through stack frames, so the parser
reads `at Context.<anonymous> (test/unit/foo.test.js:31:19)` and makes it
repo-relative using a new manifest field, `suite_cwd`. It finds the file a
failure was *raised* in, not the source file that broke it. Verified against
real output: it picks the two testcontainer test files and correctly ignores
both the `node_modules/` frames and `test/helpers/test-container.js`.

### Enabling it: a bug that would have stopped the loop converging

When I switched it on, I found that `create_fix_issue` wrote `Repo:` and `Files:` but no
`Base:`. Given no base, `jobd` falls back to the manifest base (`prod-testing`),
so the fix would have been cut from the wrong commit. It would also never have
been fast-forwarded, because only an *explicit* base advances. Verify re-runs against the
integration branch, which would never have received the fix. Both rounds would
have burned GPU jobs on branches nothing merges, and the loop would always have
ended `blocked`. The unit tests mocked the API and never followed a fix issue
through `jobd`, so they missed it. Fixed: the fix issue now carries
`Base: <integration branch>` from the verify job (backup
`z440-verify.bak-prebasefix`).

Other setup on switch-on: Verity env gained `Z440_LOOPBACK=1`,
`Z440_EXEC_AGENT_ID` (Zed), `Z440_MAX_ROUNDS=2`, `Z440_MAX_FIX_ISSUES=4`. SOR-5
(the Phase 4 red-path test, deliberately `exit 4` and still queued for Verity)
was cancelled, so no Verity wake without a bound issue could pick it up from the queue.

### First live goal with loop-back on (SOR-10, 2026-09-11)

One unit: fix `normalise()` in `question-fingerprint.js`, which trimmed before
stripping punctuation, so `"What is X ?"` normalised to `"what is x "` and
fingerprinted differently from `"What is X?"`. Dispatched 11:50:47. The goal was
`in_review` by 11:51:55. `2 insertions(+), 2 deletions(-)`, and the fix is correct
(checked by running old and new `normalise()` side by side). Suite: 507 passing,
9 pending. Issue count stayed 12 (goal, unit, verify), no fix issues, one
`jobd` at a time, no stale verify worktrees, and `prod-testing` has not moved since the clone.

### The 403 the plan feared does not happen

Plan 5.2 flags loop-back writes as likely to fail with
`cross_issue_influence_run_context_required` — the failure that cost four GPU
jobs during the original build — and says to verify before building. Verified,
from source, and it is fine:

```ts
// server/src/routes/issues.ts — dependency wakeup enqueue
contextSnapshot: { issueId: input.dependentIssueId, taskId: input.dependentIssueId, ... }
```

```ts
// server/src/services/cross-issue-influence-limit.ts
for (const candidate of [context.issueId, context.taskId]) { ... }
```

A dependency wakeup sets **both** keys, and `readRunSourceIssueId` reads exactly
those two. So the run is bound, `PAPERCLIP_TASK_ID` is populated, and the
loop-back writes attribute correctly. This is the opposite of the manual
`agent wake` path, which binds nothing — the distinction the original build
learned the hard way still holds.

---

## Phase 6 — onboarding

`/home/agent/etc/repos/<name>.json`, resolved by `jobd` as **job value →
manifest → built-in default**. A missing manifest degrades to exactly the
pre-manifest behaviour, which is the regression guard. `verifyd` reads `tests`
and `suite_cwd` from the same file.

Sorbit's manifest carries `base`, the `SKIP_TESTCONTAINERS=1` test target,
`setup`, `node_modules`, `suite_cwd`, and `max_file_bytes`.

**`max_file_bytes` is 250000, not the plan's 70000.** 70 KB came from the 32K
window; `Z440.md` already measured that at 64K every real code file in the
repo fits, including a 222 KB Dart file at ~41k tokens. The check runs before
the GPU lock, so an oversized file costs nothing rather than an hour.

### `repo-add`

Built as specified, with the secret gate **run rather than recommended**. The
plan prints the three checks from 6.3 and asks a human to run them; a check
nobody remembers to run is not a control, so `repo-add` runs all three itself
and prints `HIT` lines or `(clean)`. It still refuses to mark the repo ready —
the manifest stays `UNREVIEWED` until a person clears it, because "is this repo
safe to expose to a model" and "which suite is genuinely hermetic" are not a
script's calls.

The nets were tested by planting a fake credential in a throwaway clone; both
the filename net and the content net fired. A detector that has never fired is
not a detector.

### Acceptance

- [x] `repo-add` on a fresh repo produces a working clone and a manifest stub
- [x] It refuses a name containing `/` or a leading `.`, and an existing repo
- [x] A job naming only `Repo:`/`Files:` inherits `base` and `tests` from the
      manifest — proven accidentally and convincingly: a job sent with no
      `Tests:` line ran the manifest suite, 507 passing
- [x] An issue-level `Tests:` still overrides the manifest
- [x] A repo with no manifest behaves exactly as before (`scratch`, exit 0)
- [ ] Free-space refusal — the `df` gate is in place but was not exercised;
      testing it means filling a 231 GB disk

---

## Phase 3 — built, without the acceptance gate

The plan's 3.2 routes decomposition through
`POST /api/issues/:id/accepted-plan-decompositions`, which requires a plan
document revision that a **human has accepted**. That gate was dropped by
decision: the orchestrator is the intelligent component and is expected to clear
that bar itself. What replaces it is scope discipline in Glorp's instructions
plus a unit cap in the tooling.

`~/harness/bin/goal-fanout <goal-issue-id> <plan-file>` parses a plan file,
creates the integration branch via `create-base`, creates one child issue per
unit assigned to Zed (filling in `Base:` and `Files:`), and creates a verify
child assigned to Verity **blocked by every unit**. That last edge is the whole
mechanism: it is what the dependency wakeup fires on.

Everything is validated before anything is created, and it refuses more than 12
units without `--force`.

### The plan is wrong about the cross-issue cap

Plan 5.2 says `CROSS_ISSUE_INFLUENCE_LIMIT = 20` bounds how many children a
round can create. It does not. `assertCrossIssueInfluenceWithinRunCap` is called
only for `update`, `comment` and `interaction_resolution` on existing issues —
**there is no call site on issue creation**, and same-issue writes return `null`
before the counter is even consulted. So Paperclip places no bound at all on
fan-out width. The `MAX_UNITS` cap in `goal-fanout` and the scope rules in
Glorp's instructions are the only two things standing between a plan and fifty
queued jobs.

### Units complete; they are not reviewed

The blocker that made the loop impossible to close:

```ts
// server/src/services/issues.ts:1307
// Only done blockers resolve dependents; cancelled blockers stay unresolved
if (row.blockerStatus !== "done") { ... isDependencyReady = false }
```

`z440-exec` ended a green unit at `in_review`, so the verify child never
unblocked and the dependency wakeup never fired. Every unit needed a human to
close it — precisely what the fan-out exists to avoid.

Green units now go to **`done`**. This is not a policy setting: the issues carry
`executionPolicy: null` and always did, with no review stages to remove. The
`in_review` was chosen in the bridge code. `done` is not subject to the
in_review disposition guard (`routes/issues.ts:3698` is scoped to `in_review`
alone), so it needs no named human assignee.

`z440-verify` correspondingly completes the verify child and **hands the goal
issue** to a human — `in_review`, human assignee, agent stood down. That is the
one human touchpoint in the system. A cross-issue write, and a legal one,
because the dependency wakeup binds the run.

### Verified end to end, autonomously

One goal, two units, nobody touching anything:

```
13:58:38  goal-fanout: base agent/goal-shared-docs, 2 units, verify blocked by 2
13:59:31  u1 in_progress  u2 todo         verify blocked  goal todo
13:59:54  u1 done         u2 in_progress  verify blocked  goal todo
14:00:42  u1 done         u2 done         verify in_progress
14:01:06  u1 done         u2 done         verify done     goal in_review
```

The integration branch carried both commits; `prod-testing` stayed at
`fe505ada`.

### And the first autonomous run produced a bad diff

Worth recording, because it is the cost of the design rather than a defect in
it. Unit 2's task said "add a JSDoc block comment… do not change any executable
code". The model obeyed that literally and **deleted a twelve-line design
rationale** — the numbered principles explaining why the prompt builder is
shaped the way it is — replacing it with two generic lines. 3 insertions, 15
deletions.

The suite passed, because comments do not affect assertions. Verify passed. The
goal was handed back labelled verified.

**It was the model's own decision, and it narrated it.** From the job log:

> The file already has a JSDoc block at the top describing the module. However,
> the request asks for documentation of the prompt builders module, which should
> be more specific about what this module provides. I'll add a more detailed
> JSDoc comment at the very beginning of the file.

It then emitted a SEARCH block consuming the old comment. So it saw the comment,
judged it did not match the request, said "add", and did a replace. Not aider,
not the harness — Qwen.

In fairness the instruction was ambiguous: the old block documented a *function*,
not the module, and the task asked for a block "at the very top of this file"
where one already sat. The model resolved that collision by swapping. `diff`
edit format pulls the same way — inserting *above* an existing block means
carrying that block through the SEARCH and re-emitting it intact, which is
strictly more work than replacing it.

### Fixed in two places, and verified

**A floor under every job.** `/home/agent/etc/CONVENTIONS.md`, wired into
`.aider.conf.yml` as `read:`, so it is in the context of every edit regardless
of what the task says. It forbids removing or rewriting a comment the task did
not name, requires the old text to be carried through the REPLACE side
byte-for-byte, and says that if the task looks impossible without deleting
something, do the additive part and say so rather than resolving the conflict by
replacing.

**Re-ran the identical task** — same text, same model, same file, conventions
the only difference:

```
before:  1 file changed, 2 insertions(+), 12 deletions(-)
after:   1 file changed, 4 insertions(+)
```

The new comment went in above the original, and the twelve lines of design
rationale survived untouched.

**And a rule in Glorp's instructions** to name what must survive, not only what
must not break, with this failure as the worked example.

Two consequences still worth holding onto:

- **The end-of-goal review is now load-bearing.** It is the only place a
  content regression is caught, and it is reading a diff, not a test result.
- **Neither fix is a guarantee.** Both are prompt-level, against a 3B-active
  model. They demonstrably moved this case from wrong to right, but the class of
  failure — obeying the letter of a task while destroying something valuable it
  did not mention — is not one tests can catch.

## What is left

The plan's stated risk was that the decomposition route might reject a `process`
agent. **It does not.** The gate is `assertAgentIssueMutationAllowed`
(`server/src/routes/issues.ts:4140`), which is assignee-ownership based:

- the actor must be an agent with an `agentId`;
- `decideIssueAccess` must allow `issue:mutate`;
- the issue must be unassigned, or assigned to the calling agent.

There is **no adapter check anywhere in that path** — adapter type is
irrelevant, and being the assignee of the source issue is what matters. Moot in
the end, since the acceptance gate was dropped, but worth recording: the plan's
stated risk was unfounded.

One smaller correction: the manifest removes `Tests:` from every issue, but not
`Base:`. The integration branch is per-*goal*, and a per-*repo* manifest cannot
know it, so `goal-fanout` writes `Base:` onto every unit.

---

## Test artifacts left behind

Not cleaned up automatically, because deleting things off a live board is worse
than leaving them:

- Branch `agent/goal-smoke` on Sorbit (2 commits) plus several `agent/*` unit
  branches. Unmerged, so `agent-prune` will not touch them.
- Paperclip issue SOR-4, "Verify integration branch agent/goal-smoke"
  (`in_review`, assigned `local-board`). SOR-5, "Verify (red path test)", was
  cancelled at loop-back switch-on.
- `.aider*` and `node_modules` in Sorbit's `.git/info/exclude` — intentional,
  leave them.

## Still open

- ~~**Flutter (plan 1.4) not installed.**~~ Installed 2026-09-14 (3.47.4,
  `/home/agent/sdk/flutter`). `flutter analyze` runs on touched Dart files, but
  the 115 `*_test.dart` files still do not run: no `flutter test` yet.
- **Disk:** 25 GB free of 231 GB (2026-09-18), down from 43 GB. `jobd` and
  `verifyd` have no free-space check; only `repo-add` does.
- **Integration/postgres/live suites still cannot run**, by design — they need
  docker, and the `docker` group is root-equivalent on this box.
- ~~**Phase 5's red path has not run live.**~~ It has: loop-back fix issues
  SOR-39/40 (`termsOfUse.dart`, rounds 1 and 2), SOR-42 and SOR-45
  (`router.dart`). Worst case is still 8 fix issues (4 × 2 rounds), then `blocked`.
- **`Z440_DEFAULT_REPO=sorbit`** is set on both Zed and Verity. Per plan 6.6,
  drop it and make `Repo:` mandatory as soon as there is a second repo.

## Since this was written

Changes made after this document, recorded here only as pointers. Each one's
backup file on the Z440 or mini PC marks when it landed.

- **Dart analysis (2026-09-14).** The manifest gained `dart_analyze`. `jobd` and
  `verifyd` run `flutter analyze` on the Dart files a unit or goal touched;
  informational lints are non-fatal. Loop-back parses analyzer errors as
  failing files, and since SOR-46 its fix issues lead with the error-severity
  findings.
- **Syntax checks (2026-09-14).** The manifest's `syntax_checks` maps `js`, `html`,
  `css` and `sql` to checkers in `/home/agent/tools/`. `verifyd` runs them on
  touched files of each type.
- **Retries carry feedback.** A failed unit no longer just waits for the next
  wakeup. `z440-exec` schedules an issue-bound retry through the issue's
  `executionPolicy` and appends the previous failure to the task. The 3-strike
  rule still applies, and a board comment can reset the cycle.
- **Sentinel** (DeepSeek, same launcher as Glorp) reviews Paperclip's
  high-churn watchdog issues (e.g. SOR-43).
- **Database Operator:** a non-LLM `process` agent for reviewed, named live-DB
  runbooks (`~/harness/db-operator/README.md`).
- **Glorp moved to DeepSeek** on 2026-09-11 (`MINIPC.md`).
- **Sampling fix (2026-09-18).** Aider had been sending `temperature: 0`, so every
  unit was decoded greedily (`Z440.md`).
- **Z440 base sync (2026-09-18).** The Z440 has no GitHub credential, and
  nothing had ever refreshed its clone: `prod-testing` sat at `fe505ada` while
  GitHub was 84 commits ahead at `bd951831`. Glorp planned against the fresh
  infra-reader mirror, but Zed edited the stale tree, so a goal touching new code
  (the corporate documents feature) could not have landed. Now
  `infra-reader-sync.service` has `OnSuccess=z440-sorbit-push.service`, which
  bundles the missing commits from the mirror and sends them over a new
  forced-command key (`/etc/z440-sync/key`) to `/home/agent/bin/syncd`. syncd
  fast-forwards only, takes the GPU and verify locks non-blocking (a busy Z440
  is retried at the next sync), and re-runs `npm ci` / `flutter pub get` when
  their lockfiles moved; a failed install is retried via
  `.git/syncd-setup-from`. Sources and installer: `~/harness/z440-sync/`.
  Deliberately no second GitHub App on the Z440: the `agent` account runs
  model-written code, so a token there would be reachable by it.
- **Aider auto-lint rewrote valid SQL (2026-09-18, SOR-63).** Aider's default
  tree-sitter lint flags valid Postgres (`BETWEEN` inside `CHECK`,
  `DEFAULT CURRENT_DATE`). With `yes-always` it asked the model to "fix" it,
  three rounds, each auto-committed: out-of-scope edits to `01-schema.sql`, one
  of which (`DEFAULT 'CURRENT_DATE'`) broke schema init. The per-unit gate
  never saw it (unit suite only). `.aider.conf.yml` now sets
  `lint-cmd: sql: node /home/agent/tools/check-sql.js`, the libpg-query
  checker verifyd already trusts (backup `.aider.conf.yml.bak-presqllint`).
  Tested through aider's own `Linter`: the default flags the untouched schema,
  the new command passes it and still rejects a broken file.
- **Code and its tests in separate units (2026-09-18, SOR-63).** Unit 1 created
  a six-function module with two functions missing and went green, because
  nothing tested it yet; the next unit was told to fit the tests to the module.
  A corrective unit given the module verbatim also deviated (returned
  `[Buffer]`, dropped `ignoreNotFound`). Glorp's instructions now require code
  and its tests in one unit, forbid bending tests to code, and ask it to list
  units no test reaches. Backup `~/harness/state/glorp-AGENTS.md.bak-20260918`.
- **Goal disposition after dispatch.** Glorp's run ended with the goal
  `in_progress` and no blocker; Paperclip's recovery escalated it to a bare
  `blocked`. `goal-fanout` now parks the goal `blocked` by its verify issue.
  On green, `z440-verify` first clears the goal's blockers, then hands it to
  the human, and only then closes verify, so no dependency wakeup reaches
  Glorp (backups `*.bak-prehandoff`). The clearing write is needed: Paperclip
  refuses an agent's status change on a `blocked` issue while any blocker is
  unresolved (409, judged on the pre-write state), and a write that changes no
  status is exempt. The first version skipped it and SOR-63's handback failed.
- **Units must change every file they name (2026-09-21).** SOR-63's combined
  unit was told to edit a module and create its test file; it edited only the
  module and went green, since the suite never saw the missing tests. `jobd`
  now fails a unit (exit 6) when any file in its `Files:` list is unchanged in
  the unit's own commits, and puts an `UNDER-APPLIED` line naming the files at
  the end of the log, where the retry feedback picks it up (backup
  `jobd.bak-precoverage`). Tested live on `scratch`: a unit naming two files
  and editing one exits 6; one naming and editing one file exits 0.
- **Unit tasks were truncated to 1200 characters (fixed 2026-09-21).** When Zed's
  run was not bound to an issue, `z440-exec` took the unit from Paperclip's list
  endpoint, which cuts descriptions at `ISSUE_LIST_DESCRIPTION_MAX_CHARS` (1200),
  and sent that to the model as its task. Glorp found it on SOR-83: the landed
  helper stopped exactly at character 1200, before its `module.exports` line and
  the test-file spec. 29 past Zed units had descriptions over 1200 characters;
  any that ran unbound got a clipped task, which may account for several
  "under-applied" results (SOR-65, SOR-73, SOR-80). `pick_issue()` now re-reads
  a queue-picked issue by id and refuses to dispatch if that fails (backup
  `z440-exec.bak-prefulldesc`). Verified: the task sent for SOR-83 went from
  1068 to 2278 characters.
- **Do not un-assign and re-assign a unit to wake Zed.** Units block their
  verify issue, and Paperclip's orphan-blocker recovery hands an unassigned
  blocker to the agent that created it (Glorp) within the same second (SOR-83,
  2026-09-21). Wake an agent on an issue with
  `POST /api/agents/<id>/wakeup {"payload":{"issueId":"<id>"}}` instead. To reset
  a unit's three strikes, a comment must contain `**z440 retry cycle reset**`;
  comment text never reaches the model, so guidance goes in the description.
- **A refused unit now says so (2026-09-21).** When a unit with three strikes
  was woken again without a reset, `z440-exec` logged "leaving for human
  review" and returned, leaving the unit `in_progress` with no run until
  Paperclip's continuation recovery blocked it with no explanation (SOR-83,
  twice). It now puts the unit back in `blocked` with the board unblock
  descriptor and a comment explaining how to reset it (backup
  `z440-exec.bak-prereblock`).
- **End-to-end check (SOR-82, 2026-09-21), all fixes above in place.** Two
  units (helper, then its tests), verify green at 511 tests, goal handed to
  `local-board` with no agent assignee and no blockers, and no planner wake
  afterwards. The handback fix was also confirmed on a real goal: SOR-79's
  verify at 14:54 hit the 409 on the old code, and its re-verify at 15:05, on
  the new code, handed over cleanly.
- **Sentinel reviews a unit after its third failure (2026-09-21).** Instead of
  going straight to a human, a unit whose third attempt fails its checks is
  assigned to Sentinel with a `**z440 sentinel escalation**` comment. Sentinel
  diagnoses the three failure reports, rewrites the unit's description (the
  only text the model sees) and hands it back to Zed with a retry-cycle reset,
  or blocks it for the board if a rewrite cannot fix it. One pass per unit: a
  second three-strike cycle blocks for a human. Infrastructure errors (Z440
  unreachable) never escalate. Tested live on SOR-89: three red attempts,
  Sentinel found the task's `Files:`/"no other file" contradiction and the
  Jest-vs-chai mistake, rewrote it, and Zed passed first time. Sentinel's
  procedure is in its instructions (backup
  `~/harness/state/sentinel-AGENTS.md.bak-20260921`); `z440-exec` backup
  `z440-exec.bak-presentinel`.
- **Green goals are pushed and opened as PRs (2026-09-21).** On green,
  `z440-verify` asks verifyd for an `export` (a bundle of `agent/goal-*` beyond
  the manifest base, so the Z440 still holds no credential), and posts it to
  `infra-reader`'s `/v1/github/publish`. That route takes only a separate
  publish key (`/etc/infra-reader/publish-key`, copy in
  `~charlie/.config/infra-publish/key`; Glorp can read neither and its key gets
  401). It pushes only `agent/goal-*`, never force-pushes, opens or reuses a
  PR into `prod-testing`, and refuses branches touching `.github/` (PR
  workflows run with repo secrets). The PR link goes in the handback comment;
  a failed publish never blocks the handback. Tested against a fake GitHub
  (`infra-reader/test/test-publish.sh`, 8 cases) and live up to GitHub's write
  check. `glorp-sync` was upgraded to contents and pull-requests write the same
  day, and a live publish opened PR #461 (2 files, head
  `agent/goal-format-bytes-helper`, base `prod-testing`), which was then closed
  and its branch deleted. Backups `*.bak-prepublish`, `verifyd.bak-preexport`.
  `Z440_PUBLISH=0` on Verity turns it off.
- **Branch guard in place of GitHub branch protection (2026-09-21).** Branch
  protection needs a paid plan, and the App now has contents write. Every
  publisher push runs through a root-owned `pre-push` hook
  (`/usr/local/lib/infra-reader/hooks/pre-push`, passed as `core.hooksPath` on
  the push itself, so the service user can neither edit nor skip it). It
  refuses any push that writes a ref other than `refs/heads/agent/goal-*`,
  any deletion, and any push at all while a local copy of `prod`,
  `pre-testing`, `prod-testing` or `master` differs from the remote. The
  publisher refuses to push if the hook is missing. Glorp itself has no push
  path: no GitHub credential, no internet route, and a 401 on the publish
  route. `test-publish.sh` covers it (15 checks, including a write to `prod`
  with the publisher's own checks bypassed). The guard only covers pushes; the
  App key could still change those branches through the GitHub API, so the key
  stays readable by root and the `infrareader` service only.
- **Verity had the same 1200-character truncation** on its queue path; fixed
  the same way (`z440-verify.bak-prefulldesc`).
