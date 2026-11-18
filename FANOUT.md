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
- **The coverage gate accepted empty files (2026-09-28, found reviewing SOR-89).**
  The SOR-63 gate tested `git diff --name-only "$START...HEAD"`, so a named file
  counted as applied merely by appearing in the diff. A zero-byte create
  satisfied that. SOR-89's own winning attempt proves the path is reachable:
  aider committed `164b584c` with both named files at `0 insertions, 0 deletions`
  and only filled them 4s later in `84fefa38`. Had it stopped after the first
  commit the unit would have gone green — the gate passes, and so does the suite,
  because `cloud-run` runs bare `mocha test/**/*.test.js` with no `.mocharc` and
  no `--forbid-pending`, so an empty spec contributes no assertions and the empty
  module is never required. That is the SOR-63 false-green class still open.
  `jobd` now takes `--numstat` per named pathspec and fails the unit (exit 6)
  unless that file's own commits add or delete at least one line; binary files
  (`-` counts) and deletions still pass, and the `UNDER-APPLIED` line now
  separates "Not changed at all" from "Committed empty, with no content".
  Verified on a throwaway repo across content / empty / deletion / binary /
  missing / mixed, and against the real commits: `164b584c` gives `0 0` for both
  files and now fails, `84fefa38` gives `10 0` and `14 0` and still passes.
  Backup `jobd.bak-preemptycontent`.
- **Strike counting matched failure markers anywhere in any comment
  (2026-09-28).** `z440-exec.strikes()` counted a comment as a strike if a
  failure marker appeared anywhere in its body. Sentinel is an LLM writing prose
  *about* those failures, so its diagnosis could be counted as a fresh strike and
  silently spend the one cycle its rewrite was meant to buy. SOR-89 was safe only
  because Sentinel happened to put the reset marker and the diagnosis in a single
  comment (the reset index is excluded from the count), which nothing enforces.
  Both bridges now match the marker at the *start* of the body and require the
  comment to be the agent's own; unknown authorship still counts, so a miscount
  can only stop early rather than retry on the GPU forever. `z440-verify` had the
  same defect and looser: it matched the unbolded substring `z440 verify failed`
  across *all* comments with no reset scoping, so a human writing that a verify
  had failed consumed an infrastructure attempt and could park a goal `blocked`
  early; its marker is now the constant `FAILURE_MARKER`, posted and matched from
  one place. Tested against SOR-89's real comment history (unchanged: 3 strikes
  before the reset, 0 after) plus the two Sentinel-prose shapes and a
  genuine-failures-after-reset case. Backups `*.bak-prestrikeauthor`.
- **Retry latency is the scheduler sweep, not the work (2026-09-28).**
  `retry_monitor()` set the due time 10s out, but Paperclip's monitor tick rides
  the heartbeat scheduler (`HEARTBEAT_SCHEDULER_INTERVAL_MS`, default 30000,
  floored at 10000 in `config.ts`), so each of SOR-89's retries waited ~29s for
  ~25s of work. The 10s was never a safety margin: `report()` releases the
  checkout *before* the board write that schedules the monitor, so the checkout is
  already gone when a due time first exists. Now 2s, which stops a long offset
  from missing the next sweep. The sweep itself was then cut to the 10000ms floor
  via a systemd drop-in on `paperclipai.service` (`MINIPC.md`), which also speeds
  up every agent's timer heartbeat, not just unit retries. Restarted clean on an
  idle board: banner `Heartbeat enabled (10000ms)`, health 200, no errors after
  startup. Expect a retry to wake in ~2-12s instead of ~29s.
- **`agent-prune` never pruned a branch checked out in a worktree (2026-09-28).**
  `git branch --merged --list` marks the current branch `* ` and one checked out
  in another worktree `+ `. The loop piped that through `tr -d ' '`, which left
  the `+` attached, so it ran `git branch -d +agent/...` and failed every night
  with `error: branch '+agent/20260921-150256-17508' not found`. It now reads
  `--format='%(refname:short)'` and skips branches a live worktree holds with a
  reason, leaving them for a later run once the worktree ages out. Tested on a
  throwaway repo: prunes a free merged branch, skips an in-use merged branch
  quietly, never touches an unmerged one. Backup
  `agent-prune.bak-preworktreebranch`. Deployed and run live: the nightly error
  is replaced by `skipped .../agent/20260921-150256-17508 (checked out in a
  worktree)`, with the branch and all 103 `agent/*` refs intact.
- **A retried unit keeps a `triggered` monitor after it goes `done`
  (2026-09-28, not fixed).** SOR-89, SOR-83 and SOR-65 — every unit that used a
  retry — end `done` with `executionState.monitor.status: "triggered"` and
  `clearedAt: null`. It is inert, not a pending wake: the trigger already nulled
  `monitorNextCheckAt` and stripped `executionPolicy`, and the dispatch query
  requires `monitorNextCheckAt is not null`, so it cannot fire again even if a
  human reopens the unit. Nor can the harness clear it. Paperclip rebuilds the
  state from the `monitor*` columns in `derivePersistedMonitorState`, which
  returns `triggered` whenever `lastTriggeredAt` or `attemptCount > 0`; the two
  branches that write a `cleared` state both require a monitor still present in
  the policy, which the trigger has already consumed, and any `executionState` an
  agent submits is overwritten by that derived state. Confirmed present in the
  deployed build (`~/.paperclip/.../dist/services/issue-execution-policy.js`), so
  this is upstream display/audit behaviour, not a harness defect. Left alone
  rather than patching a managed install.
- **A unit's assignment wake can be consumed by a run that works on a different
  unit, stranding it `in_progress` (2026-09-30; sweeper deployed).** SOR-97 (unit 4 of the settings-page goal) sat `in_progress` with
  no checkout, no run and no monitor from 12:06 to 13:01 while Zed idled, and
  verify stayed blocked behind it. Zed's run `f82ebb1d` was created at 11:58:41,
  when SOR-97 was assigned, and its wake payload still names SOR-97. It started
  at 12:06:48 as the retry of SOR-96's first failed attempt: `retryOfRunId` is
  the failed run and its `taskId` is SOR-96. Paperclip checked SOR-97 out from
  the wake payload; `z440-exec` follows `PAPERCLIP_TASK_ID`, so it dispatched
  SOR-96 and, on exit, released and left SOR-97 stranded. Zed wakes on
  assignment only, so nothing woke it again. `report()` cannot fix this: it
  cleans up claims `z440-exec` made, and this one it never did. Recovered by hand
  with a targeted `POST /api/agents/<zed>/wakeup` carrying the issue id (run
  `09519f9f` checked it out and finished in ~1 min).
  **Not proven.** The merge is read from the run records only. Two attempts to
  reproduce it on `scratch` failed: a failing unit with one passing unit queued
  behind it, and with three queued behind it. In both, the retry became its own
  `automation` run (`retryOfRunId` set, ahead of the queued units) and every unit
  finished correctly. What differed in the real event is unknown; candidates are
  the ~8 min the wake waited in the queue, the 4m22s first attempt, and timing.
  Repro issues SOR-99 to SOR-104 and their unmerged `agent/*` branches on
  `scratch` remain.
  **Fix: `~/harness/bin/z440-stale-sweeper`** (new file, no backup). Wakes Zed for
  a unit that is assigned to it, `in_progress`, with `checkoutRunId`,
  `executionRunId` and `monitorNextCheckAt` all null, no queued/running Zed run
  bound to it, and untouched for `STALE_MINUTES` (10). It fires only when Zed has
  no queued or running run at all, because a wakeup made while the agent is busy
  is coalesced and lost (the same behaviour `retry_monitor()` documents). At most
  `SWEEP_MAX_WAKES` (2) wakes per issue, one per sweep, and it never touches
  status, assignee, comments or checkout, so `z440-exec`'s three-strike refusal
  still bounds GPU cost. It does nothing if Paperclip or a listing fails.
  Verified against a fake Paperclip: stranded is woken; recent, checked out,
  monitor pending, run bound, other assignee are left alone; a busy agent
  defers; `--dry-run` sends nothing; the cap stops a third wake; a down or
  non-loopback API is a no-op. A live `--dry-run` on the idle board correctly
  found nothing. **Never exercised against a real stranded unit**, because none
  could be fabricated safely: the only live check of the wake call is the manual
  SOR-97 recovery. Unit files `z440-stale-sweeper.{service,timer}` (5-minute
  timer, log `~/harness/logs/stale-sweeper.log`, state
  `~/harness/state/stale-sweeper.json`) were installed and enabled by Charlie the
  same day; the first sweep logged `no in_progress units assigned to the agent`
  and the timer's next run was 5 min out. It has not yet fired on a stranded unit. Worst-case
  detection is ~15 min (10 min grace plus one 5-min tick). It covers Zed only;
  Verity units can strand the same way and are not swept.

- **2026-10-05 — Code-quality prompt rules for Glorp and Zed** (full evidence in
  [`QUALITY-EVAL.md`](QUALITY-EVAL.md)). Five goals on `scratch` (SOR-105, 108,
  111, 114, 117) showed that the **planner**, not the coder, decides the quality of
  code it embeds in a unit (the "new file under ~60 lines: paste it" rule): Qwen
  copied `notes.py` and `orders.py` byte for byte, including a query-per-row loop
  in both. Behaviour tests passed; `radon`/`bandit`/`lizard` were clean. Fix at
  the writer: Glorp's `AGENTS.md` gained "The code you embed is your code" (no
  query in a row loop, aggregate in SQL, parameterised SQL, neutralise CSV/HTML
  output, one transaction, embedded tests are exact text and must `py_compile`),
  then a mandatory "Code review" section in the dispatch comment. Proved by an
  exact-spec A/B: the same spec produced N+1 under the old prompt and 1 SQL
  statement (measured with `set_trace_callback`) under the new one, twice; CSV
  formula cells were unescaped until the second change. Zed's `CONVENTIONS.md`
  gained "Text the task gives you verbatim". **That rule did not fix** Zed
  altering one test line in the same place in two runs; the cause is unknown, and
  the per-unit gate that would catch it does not exist on `scratch` (no manifest),
  so whether a retry repairs it is **not verified**. No reviewer agents were
  added. Backups: `~/harness/state/glorp-AGENTS.md.bak-20261005` (before change
  1), `…bak-20261005b` (before change 2),
  `/home/agent/etc/CONVENTIONS.md.bak-prequalityrules`. 5 runs of toy
  Python/SQLite modules; nothing run on Sorbit.

- **2026-10-05 — `scratch` manifest added** (`/home/agent/etc/repos/scratch.json`,
  new file, no backup). `tests`: `PYTHONDONTWRITEBYTECODE=1 python3 -m pytest -q
  -p no:cacheprovider`; no `base`. Before this `scratch` had no manifest, so
  per-unit gates ran nothing (a unit reported success with a failing test) and
  every verify went red with "no suite". Proved with `verifyd` on two existing
  branches: `agent/goal-invoices-module` exit 0 (20 passed),
  `agent/goal-purchases-module` exit 1 naming the altered test. Two live goals
  (SOR-120, SOR-121) then went unit → verify green → `in_review`, first attempt,
  no strikes. **Side effect:** `scratch` is no longer the "no manifest behaves as
  before" regression guard (see the 2026-09 manifest entry); use another
  throwaway repo for that. **Not verified:** the retry path; no unit failed, so
  it was not exercised. See [`QUALITY-EVAL.md`](QUALITY-EVAL.md).

- **2026-10-05 — Retry path exercised live for the first time on a real gate
  failure** (SOR-130, `scratch`; no harness change). Hand-written plan via
  `goal-fanout` naming a third file the task never mentions. Attempts 1–3 each
  `exit 6` (`UNDER-APPLIED … Not changed at all: WORDCOUNT_NOTES.md`), retry
  monitor fired each time, strike 3 posted the escalation, **Sentinel diagnosed it
  correctly and rewrote the task** with full contents of all three files, the
  next attempt passed, verify went green, goal `in_review`. A plain retry did not
  fix it (identical failures); the Sentinel rewrite did. Also found:
  `paperclipai issue create --description` converts a literal `\n` to a real
  newline (reproduced; `\t` survives), which corrupts embedded test text with
  escapes in it. The `child:create` path `goal-fanout` uses showed the same
  symptom (Sentinel saw a raw newline in a string literal in attempt 1) but the
  exact conversion point there was **not** traced. Not fixed. Three kept
  worktrees from the failed attempts remain in `/home/agent/work` until
  `agent-prune`. The earlier run-2/4 test mutation was not reproduced here, so
  whether a retry repairs *that* is still unverified. See
  [`QUALITY-EVAL.md`](QUALITY-EVAL.md).

- **2026-10-16 — Mini PC ↔ Z440 moved from the router to a direct ethernet
  cable.** No harness change; a mini-PC network change only. After the move
  `enp3s0` had link (1 Gb/s) but no IPv4, so `192.168.1.199` gave *No route to
  host*: `z440-tunnel` crash-looped (41 restarts) and the 11:21 `z440-sorbit-push`
  failed. The Z440 was found via IPv6 link-local (`ff02::1` ping on `enp3s0`);
  its ed25519 host key matched the `known_hosts` entry for `.199`. **Fix:** kept
  the address, so the SSH aliases, tunnel unit, glorp nft allow-rule and sync
  scripts are untouched. On the mini PC, `nmcli con mod "Wired connection 1"
  ipv4.method manual ipv4.addresses 192.168.1.121/32 ipv4.routes
  192.168.1.199/32` (route confirmed with `ip route get`, rtt ~0.3 ms). **Proved:**
  `ssh z440`, tunnel active and `localhost:8080/health` ok, `z440-exec --selftest`
  (`z440-job` reachable), glorp's `z440-job` ssh returns JSON, GPU lock free.
  **Not verified:** a real goal end to end, and a fresh `z440-sorbit-push` run
  (the one failure was during the outage); the route surviving a reboot (the
  profile is saved, not rebooted). **Side effects:** the Z440's default gateway
  `192.168.1.1` and DNS are now unreachable, so it has no internet (consistent
  with LAN-only, but its GitHub Actions runners, not part of the harness, are
  cut off); with no NTP source its clock reports "not synchronized" and may
  drift (it matched the mini PC at 11:31 BST). Three `/home/agent/work`
  worktrees from 2026-10-05 remain for `agent-prune`. Backup: none (new NM
  settings; previous profile was DHCP `auto`).

### 2026-10-16 — GitHub Actions runners removed from the Z440

CI now runs on GitHub-hosted `ubuntu-latest`. The six self-hosted runners
(`sorbit-local-1..6`, repo-level on `Sorbit-ltd/Sorbit`) were already offline
after the direct-cable move (no internet), and today's nightly sat queued
behind them. They were never part of the harness.

- **Proved on hosted before switching:** `workflow_dispatch` with
  `runner=github-hosted` on `prod-testing`: PR Gate (build-check, cloud-run
  lint/unit/smoke/integration, e2e), Flutter E2E, Anthropic contract, both
  guards — all green. After the default flip (Sorbit PR #485, merged 11:37 UTC)
  the push-triggered Flutter E2E passed with no dispatch input.
- **Changed:** PR #485 flips every `runner` default / `|| 'self-hosted'`
  fallback to `github-hosted`; `self-hosted` is still a selectable input but
  nothing serves it. The runners were deleted from GitHub, units stopped and
  uninstalled (`svc.sh`), and `~/actions-runner-{1..6}`, `~/runner-admit.sh`,
  `~/setup-runners.sh`, `/tmp/runner-admit.log` deleted. Disk 24 GB → 49 GB free.
- **Not verified:** `prod-deploy` on hosted (live deploy; run once supervised).
  `skip-audit` fails on hosted *and* did on self-hosted (pipefail + empty grep)
  — pre-existing, not fixed. Hosted minutes: ~610 wall-clock min/month vs 2,000
  free on the private-repo free plan. The 07:55 nightly run (38870259330)
  had been queued behind the offline runners; GitHub ended it as `cancelled`.
- **Docker/toolcache cleanup (2026-10-19, by exact name):** removed the 3 buildx
  builders (+ state volumes), 2 long-exited containers, the `sorbit-cloud-run:smoke`
  image, 17 `act-*` CI scratch volumes and `~/hostedtoolcache` (runner tool
  cache). Disk 48 GB -> 66 GB free. **Kept:** `pgvector/pgvector:pg16` and
  `testcontainers/ryuk` (no internet to re-pull), `sorbit_app_postgres_data`, and
  95 anonymous volumes (~17 GB) that were not inspected — they may hold data.
- **Backup:** none. The before-state (GitHub runner list, `setup-runners.sh`,
  `runner-admit.sh`) was saved only to a temp scratchpad, which the mini PC's
  2026-10-16 14:52 reboot wiped — so it is gone. Re-registering means a fresh
  runner token and re-writing the setup by hand (runners were repo-level on
  `Sorbit-ltd/Sorbit`, as `jack` from `~/actions-runner-N`, labels
  `self-hosted,linux,sorbit-local`, plus `port-host` on #1). This also makes the "no gateway/internet" side effect in the
  2026-10-16 cable entry moot for CI; the clock is still unsynchronized.

### 2026-10-16/19 — Z440 silent on the direct cable until restarted

After the mini PC rebooted (2026-10-16 14:52) the Z440 stopped answering on
192.168.1.199 (`ARP FAILED`). The mini PC side was correct throughout: `enp3s0`
up at 1 Gb, the persistent /32 route in place (it survived the reboot, which is
the first time that was exercised), but `enp3s0` had received **0 bytes** since
boot, i.e. the Z440 sent nothing at all. The Z440 has no screen, so the state it
was in is unknown. A restart of the Z440 on 2026-10-19 fixed it: `eno1` came up
with 192.168.1.199/24 and ssh worked within seconds.

- **Cause: not established.** Candidates: the Z440 hung or stalled during boot
  after the power event, or `eno1` was not brought up. Nothing was captured.
- **After restart (checked 2026-10-19):** Paperclip `/api/health` ok; `llama.service`
  active, `:8080/health` ok, model resident (15.8 of 17.1 GB VRAM); the glorp ->
  `z440-job` forced-command path answers (an empty job returns the expected
  `job is not valid JSON`); `z440-exec --selftest` ssh check passes. Its API check
  returns 401 from an interactive shell — it needs the agent credentials Paperclip
  injects, so that part was not exercised. No full goal run or sorbit-push was done.
- **`logrotate.service` failed since boot — fixed 2026-10-19:** `/etc/logrotate.d/playit`
  pointed at a log that never existed (playit.gg game-server tunnel agent, installed
  2024-04, service disabled, unrelated to the harness). Removed with
  `apt-get purge playit` (drops the logrotate config, `/opt/playit`, the unit),
  plus `~/.config/playit_gg` (held a secret key — revoke it in the playit.gg account
  if you want it dead) and the empty `/etc/playit`. logrotate now exits 0; no failed units.
- **If it recurs:** from the mini PC, `ip -s link show enp3s0` — carrier up with
  RX 0 means the Z440 side is silent (not a mini PC/route problem). It needs a
  screen or a power cycle.

### 2026-11-04 — Z440 reached over wifi instead of the direct cable

No harness change; a mini-PC network change only. The direct ethernet cable is no
longer connected (`enp3s0` is `NO-CARRIER`); the Z440 stays at `192.168.1.199` on
`eno1`, on the same LAN as the mini PC's wifi (`wlp2s0`, `192.168.1.121/24`).
`ip route get 192.168.1.199` now shows `dev wlp2s0`, and `ssh z440` logs in
passwordless with the same `~/.ssh/z440_key`. No config, key or backup file changed.

- **Either path works.** With the cable back in, the persistent `/32` route on
  `enp3s0` is more specific than the wifi `/24` and takes over again.
- **Not verified:** only the `ssh z440` login and the Z440's addresses were checked.
  `z440-job` / `z440-verify`, `z440-tunnel` and a full goal run over wifi have not
  been exercised.

### 2026-11-04 — Laptop key added to `jack`'s authorized_keys on the Z440

So an engineer's laptop on the same LAN can SSH to the Z440 directly (not via the
mini PC). Added one `ssh-ed25519` public key (comment `jackbower835@gmail.com`) to
`jack@jack-1-0:~/.ssh/authorized_keys`, appended after the existing
`laptop-to-z440` key. It is an unrestricted key: full shell as `jack`, who has
passwordless sudo. Before-state: `~/.ssh/authorized_keys.bak-before-laptop-key` on
the Z440. To revoke, delete that key's line.

- **Not verified:** the login from the laptop itself has not been tried; only the
  file contents and permissions (`0600`) were checked.

### 2026-11-04 — Persistent GPU on/off toggle for the model, and the GPU wedge it exposed

Requested so the model can be taken off the GPU and stay off across boots. Added on
the Z440 (all new files, so there is no `.bak`; nothing existing was edited):
`/usr/local/bin/llama-gpu`, drop-ins `/etc/systemd/system/{llama,llama-warmup}.service.d/10-gpu-toggle.conf`
(`ConditionPathExists=!/etc/llama/gpu-disabled`), and
`/etc/udev/rules.d/90-gpu-no-runtime-suspend.rules`. Usage is in Z440.md. The board
was idle (no `aider`/`jobd`, `/home/agent/jobs` empty) when it was installed.

- **What broke:** the first test unloaded the model with the GPU left on
  `power/control=auto`. The idle GPU runtime-suspended and the resume failed
  (`amdgpu: PSP load kdb failed`, `resume of IP block <psp> failed -62`,
  `runtime_status=error`). `llama.service` then started with
  `no ROCm-capable device is detected` and ran on the **CPU**. Setting
  `power/control=on` did not clear it; a restart of the Z440 (by the user) did.
  The model was down for roughly five minutes.
- **Why it never showed before:** the resident model kept the GPU busy, so it never
  idled. Any future "GPU idle" state, not just this toggle, is exposed to it.
- **Fix:** the udev rule pins the card's `power/control` to `on`.
- **Proved after the restart:** the flag survived the reboot and `llama.service` did
  not start while `litellm` did; with the GPU idle (rule written, `control=on`) and
  the model off, it stayed `active`; `llama-gpu on` loaded the model onto the GPU
  (15.9 GB VRAM, health ok, no "no ROCm-capable" in the journal) and warmup ran.
  Also proved before the restart: a direct `systemctl start llama.service`, a
  `litellm` restart and `llama-warmup` all left the model off while the flag was set;
  `off` refused while the GPU lock was held.
- **Not verified:** the udev rule has been applied by `udevadm test` and by setting
  the same value by hand, but not by a boot (`control` was `auto` after the restart,
  since the rule did not yet exist). No reboot has been done with the model *on*
  since the rule, and no goal run was done. dmesg shows
  `amdgpu: failed to pin userptr` at model load; the load still succeeded and it was
  not compared with an earlier boot.

### 2026-11-16 — Remaining CI Docker leftovers removed from the Z440

Not a harness change (the harness does not use docker); disk housekeeping after the
2026-10-16 runner removal. Disk 55 GB -> 68 GB free (70% used).

- **Removed, by exact ID:** the 17 untagged images left by old CI builds, the
  `moby/buildkit:buildx-stable-1` image (its builders went on 2026-10-19), and 94
  of the 95 anonymous volumes the 2026-10-19 entry left uninspected. Inspected
  this time: 92 were Postgres data directories (`PG_VERSION` present) created
  July to September, 2 were empty. No container existed, so nothing was using them.
- **Kept:** `pgvector/pgvector:pg16` and `testcontainers/ryuk:0.14.0`, the named
  `sorbit_app_postgres_data` volume, and one anonymous volume (`8a3d4f7b…`, 192 KB,
  dated 2026-03-04) holding a pgAdmin config (`pgadmin4.db`, `azurecredentialcache`).
- **Backup:** none; the Postgres volumes were not dumped or opened beyond checking
  for `PG_VERSION`.
- **Not verified / noticed:** `llama.service` was `inactive` (`litellm` active) and
  `/etc/llama/gpu-disabled` was not checked. `/opt/playit` exists again (binaries
  dated 2026-10-29, directory 2026-11-04) although the 2026-10-19 entry records it
  purged; not investigated. Still reclaimable, untouched: `~/.cache/act` (343 MB),
  the journal (3.4 GB, no size cap) and 14 disabled snap revisions.

### 2026-11-16 — Trial of Qwen3.6-35B-A3B as Zed's model (set up, not switched)

Candidate replacement for Qwen3-Coder-30B-A3B: same 3B-active MoE shape, one
generation newer (Apache 2.0, released 2026-04). **`llama.service` is unchanged and
still the configured model; nothing was switched.** Board idle throughout (no
`in_progress`, `work/` + `jobs/` empty, GPU lock free). No existing file was edited,
so there is no `.bak`.

- **Added on the Z440:** `~/models/Qwen3.6-35B-A3B-UD-Q4_K_XL.gguf` (22.4 GB, Unsloth,
  sha256 `707a55a8…` matches Hugging Face) and `/etc/systemd/system/llama-trial.service`
  (static, never enabled; `Conflicts=llama.service llama-warmup.service`). It serves
  on the same `127.0.0.1:8080` and alias `qwen3-coder`, so litellm, aider and `jobd`
  are untouched. It does **not** carry the `gpu-disabled` condition. To run it:
  `sudo systemctl start llama-trial.service`; stop it to free the GPU. Left stopped.
- **Flags that differ from `llama.service`:** `--n-cpu-moe 17` (15.4 GB VRAM at 64K;
  20 gave 14.0 GB), `--reasoning off` (the model thinks by default), `--repeat-penalty
  1.0 --presence-penalty 0` (the card recommends presence 1.5 for non-thinking; not
  used, on the reasoning that it works against verbatim SEARCH/REPLACE text; that
  choice was not A/B tested), `--cache-ram 0 --ctx-checkpoints 0` (see below).
  Build 10743 loads the `qwen35moe` architecture without a rebuild.
- **Probes**, replayed through `ssh z440-job` with the original task text and **no
  `base`**, so each job left only a throwaway `agent/20261116-*` branch and nothing
  was fast-forwarded or published. Baseline is the *recorded* result for the current
  model, not a same-day run (the model is toggled off and the flag was left alone).

  | Probe | Qwen3-Coder (recorded) | Qwen3.6-35B-A3B |
  |---|---|---|
  | SOR-115 `purchases` unit, `scratch` | test line altered 2/2, 1 test fails | **5/5 byte-identical** to the embedded text, 19 passed |
  | SOR-8 JSDoc on `prompt-builders.js` | 12 lines deleted pre-CONVENTIONS, +4/-0 after | 3/3 additive (+5/-0, +5/-0, +4/-0) |
  | `chatpage.dart` one-line comment | 132 s wall, 533 prefill / 17.0 generate tok/s at 29.2k tokens | 100–114 s, 445–462 / 26–27 at 30.2–31.6k |
  | `teacherParentHome.dart` | 175 s, 481 / 15.4 at 40.8k | 137–141 s, 412–420 / 24.5 at 44.8k |

  Short prompts generate at ~29 tok/s (was ~20). The new tokenizer yields more tokens
  for the same file, and the files may have moved since September, so the Dart rows
  are indicative, not like for like.
- **What broke: the server wedged, twice.** With default prompt caching, the sixth
  job hung on a small follow-up request: first a 142 s "prompt eval" of 4 tokens, then
  no progress at all, one thread at 100% CPU. The kernel logged
  `amdgpu … kfd_svm: svm_range_dma_map_dev: failed -12 dma_map_page` continuously from
  the same minute (11,589 lines in the first minute). Restarting with `--cache-ram 1024
  --ctx-checkpoints 4` wedged again on the second request. With `--cache-ram 0
  --ctx-checkpoints 0` the stuck job finished and a six-job soak (2 purchases, 2 JSDoc,
  2 Dart) ran with **zero** `kfd_svm` lines. Both hangs were on a request that restored
  a context checkpoint.
- **Cause: not established.** Fits both "context-checkpoint restore path for this
  hybrid architecture on ROCm" and "host RAM pressure": this model keeps ~10 GB of
  experts in system RAM, a Minecraft server (`java -Xmx6G`, 7.3 GB RSS, not part of the
  harness) is running on the Z440, and 2.4 GB of swap was in use. Neither was isolated.
  Cost of no cache: aider's commit-message requests re-read ~6.8k tokens (~12 s each).
- **Not verified:** a goal through Paperclip (Glorp → Zed → Verity) on this model;
  retries; JS/Dart edits of any substance (the Sorbit probes are one-line or comment
  edits); anything beyond ~20 minutes of continuous serving; a reboot. Samples are
  small (5 + 3 + 4 runs). `aider` printed `Summarization failed … cannot schedule new
  futures after shutdown` at the end of the purchases runs; whether the current model
  does the same was not checked (one older log has it).
- **Residue:** throwaway branches `agent/20261116-{142843-14065,143734-31541,144031-13546,150459-27739,151025-9152}`
  on `scratch` and `agent/20261116-{143208-10593,143251-3992,143445-10645,150817-21067,150845-23461,151331-17508,151547-17060}`
  on the Sorbit checkout (unmerged, so `agent-prune` leaves them); `~/models/qwen36-download.log`.
  Disk 68 GB -> 47 GB free (79%).
- **To switch for real** (not done): point `llama.service`'s `-m` and flags at the trial
  values with a `.bak`, keep `max_input_tokens` at 65536, and run a real goal first.
