# Sorbit coder harness — showcase

A talking-points document for demoing the harness. It is written to be shown or
read aloud: what the system does, what it is built from, and — the part people
actually ask about — **how it catches AI-written code that is unsafe or wrong**.

Everything below is taken from what is deployed and from recorded runs. Where a
check has a limit, or has never fired live, this document says so. Those honest
limits are part of the pitch, not something to hide: the system is designed
around not trusting the model.

Current as of **2026-11-18**. Depth lives in [`ONBOARDING.md`](ONBOARDING.md)
(architecture and invariants), [`QUALITY-EVAL.md`](QUALITY-EVAL.md) (the
measured runs) and [`FANOUT.md`](FANOUT.md) (the dated change log).

---

## 1. The 30-second pitch

> You write a goal in plain English. A cloud planner breaks it into small,
> one-file units. A **local** coding model on a GPU box edits one unit at a
> time, and every unit has to pass tests and checks before it is accepted. When
> the whole branch is green it is pushed as a pull request — and **a human
> reviews and merges it**. No AI agent holds a credential that could merge.

Three things to land with an audience:

1. **The model that writes code is in a cage.** No network, no credentials, no
   sudo.
2. **Every change is checked before it is accepted**, and a failure is retried,
   then escalated, then handed to a human — never silently passed.
3. **It is cheap.** A typical goal costs **2–4 cents** of cloud spend. The code
   model runs on hardware we own.

---

## 2. How it works

```
 you write a goal ──▶ GLORP    plans: splits the goal into small units
                      │        (cloud model, metered, read-only access to the repo)
                      ▼
                   ZED         codes: one unit at a time
                      │        (local Qwen3-Coder on a GPU box, via aider)
                      ▼
              per-unit gates   tests · static analysis · "did you really change the file?"
                      │
          fail ──▶ retry ×3 ──▶ SENTINEL rewrites the task ──▶ retry ×3 ──▶ human
                      │
                      ▼  every unit green
                   VERITY      runs the whole suite on the integration branch
                      │        red → scoped fix units (max 2 rounds) · green → publish
                      ▼
              pull request  ──▶  HUMAN reads the diff and merges
```

Five agents, each with one job and only the access that job needs:

| Agent | Job | Brain | Can it write code? | Can it publish? |
|---|---|---|---|---|
| **Glorp** | Plans the work | DeepSeek V4.1-Flash (cloud, metered) | Only as text embedded in a task | No |
| **Zed** | Edits one unit | Qwen3-Coder-30B-A3B (local GPU) | Yes — in a sandbox | No |
| **Verity** | Runs the checks | none (deterministic) | No | Pushes a branch + opens a PR; cannot merge |
| **Sentinel** | Diagnoses a unit that failed three times | same cloud model as Glorp | No | No |
| **Database Operator** | Applies reviewed, named DB runbooks | none (fixed scripts) | No | n/a — outside the coding flow |

Speed, for scale: a two-unit goal has gone from assignment to "ready for review"
in about **2.5 minutes**; a small unit takes 20 seconds to 3 minutes. Units run
strictly one at a time (one GPU), so a 10-unit goal costs about ten times one
unit.

---

## 3. Technologies

| Layer | Technology | Why it is there |
|---|---|---|
| Orchestration / board | **Paperclip** (self-hosted, embedded Postgres) | Issues, agent assignment, wakeups, an audit trail of every run |
| Planner | **DeepSeek V4.1-Flash** via a metering proxy, run by **Claude Code** | Cheap, strong at reading a repo and decomposing work |
| Code model | **Qwen3-Coder-30B-A3B**, Q4_K_XL quant, 64K context | Runs locally; mixture-of-experts so only ~3B parameters are active per token |
| Model serving | **llama.cpp** (`llama.service`) behind **LiteLLM** | OpenAI-compatible endpoint; sampling is set server-side in one place |
| Hardware | Radeon GPU with 17.1 GB VRAM (HP Z440), plus a mini PC as the orchestrator | The model never leaves our hardware |
| Editing | **aider** (`diff` edit format) | Applies search/replace edits and commits per unit |
| Executor protocol | **SSH forced commands** (`jobd`, `verifyd`, `syncd`) | Each key can run exactly one program — the channel *is* the API |
| Checks | **mocha** (511 unit tests), **`flutter analyze`**, syntax checkers for JS / HTML / CSS / SQL | Cheap, deterministic, fast — a green verify takes ~3 seconds |
| Source control | **git** worktrees and integration branches; **GitHub** PRs via a GitHub App | One isolated worktree per job; one reviewable branch per goal |
| Cost control | **deepseek-meter** (per-request ledger and hard caps), a quota guard timer | A runaway loop hits a cap, not a bill |
| Isolation | Unprivileged Linux users, uid-based firewall, `no_new_privs`, systemd | The planner and the coder cannot reach what they should not |

Why a *local* model for the code: the code is written on a machine with no
internet and no keys, so even if the model is manipulated by text in the repo,
there is nothing for it to exfiltrate and nothing for it to publish with.

---

## 4. How it catches code that is unsafe or inaccurate

The honest framing: **no single check is trusted.** There are five layers, and
each one exists because of something that actually happened. Prevention first,
then detection, then recovery, then a human.

### Layer 1 — Contain the writer (unsafe code can't do damage)

The model's output is treated as untrusted code, because it is.

- The `agent` account that runs the model has **no sudo, no GitHub key, no
  internet**. A token there would be reachable by model-written code, so none is
  there.
- It works in a **clone**, not the owner's checkout — the owner's tree holds a
  live admin credential that a clone never copies.
- Work leaves the box only as a **git bundle**, pushed by a separate service
  with a separate key that the planner cannot read.
- The publisher is guarded by a **root-owned pre-push hook** that the service
  cannot edit or skip. It pushes only `agent/goal-*` branches, **never
  force-pushes**, opens PRs only into `prod-testing`, **refuses any branch that
  touches `.github/`** (CI workflows run with repo secrets), and refuses to push
  at all if the protected branches differ from the remote.
- The planner (Glorp) is firewalled by uid: it can reach the metering proxy, the
  board and three bounded read operations — read-only SQL (a single `SELECT`, in
  a forced read-only transaction, with row/time/output caps), fixed log queries,
  and a mirror of the repo. No arbitrary shell, no IAM, no secret values.

*Result:* the worst thing a bad generation can do is produce a bad branch that a
human then declines to merge.

### Layer 2 — Gate every unit before it is accepted (inaccurate code is stopped early)

After each unit, before it is fast-forwarded onto the integration branch:

| Gate | Catches | Failure result |
|---|---|---|
| **Unit test suite** | Code that does not do what the tests assert | Strike + retry with the failure appended |
| **`flutter analyze`** (Dart) | Type errors and analyzer errors in touched files | Strike + retry |
| **Syntax check** (JS / HTML / CSS / SQL) | Files that do not even parse | Strike + retry |
| **Coverage-of-change gate** (exit 6) | A unit that was told to change a file and didn't — or created it empty | Strike + retry |
| **Base-moved check** (exit 75) | Another change landing underneath the job | No ref updated; job refused |

Two of these are worth telling as stories because they were *found by watching
the system fail*:

- **The empty-file hole.** A zero-byte new file used to pass: the gate looked
  only at which files changed, an empty test file asserts nothing, and the empty
  module was never imported. The gate now counts added/deleted lines **per named
  file** and fails unless each one really changed.
- **The test the model rewrote.** In two recorded runs the model silently
  changed one line of a test it had been told to copy exactly, which broke that
  test. With no gate configured the unit still reported success. Once a real
  suite was attached, running the gate on that exact branch returned
  `exit 1 — 1 failed, 18 passed`, naming the test. The mutation is *detected*
  deterministically.

### Layer 3 — Verify the whole branch (individual units can be right and the sum wrong)

When the last unit lands, Verity runs the entire suite, analyzer and syntax
checks against the **integration branch**, in a fresh worktree.

- **Red:** Verity parses the failing files out of the test output and creates up
  to 4 scoped fix units for the coder, then re-verifies. **Maximum 2 rounds**,
  and a file already tried in two rounds is excluded from a third. After that the
  goal is `blocked` for a human — worst case is 8 fix units, never an endless
  loop. If no failing file can be parsed, it leaves the issue for a human rather
  than guessing.
- **Green:** publish the PR. Every report, green or red, carries a **scope
  statement** of what the green does and does not cover (see Layer 5).

### Layer 4 — Recover, don't just retry

- **Three strikes** per unit, each retry carrying the previous failure.
- On the third strike, **Sentinel** reads the three failure reports and
  *rewrites the task* rather than repeating it. In a forced-failure probe, three
  identical GPU attempts failed the same way; Sentinel diagnosed that the task
  text never described the third file, rewrote it with full contents, and the
  fourth attempt went through to a green verify. The lesson recorded: for
  failures caused by the *task text*, the rewrite is what repairs it, not the
  retry.
- Still failing after a second cycle → `blocked`, which always means "a human
  decision is wanted", never "the harness is broken".
- Infrastructure errors (GPU box unreachable) never count as strikes and never
  escalate.
- Cost bounds everywhere: 1800 s / 200-turn run limits, per-hour / per-day /
  per-month spend caps, one job on the GPU at a time.

### Layer 5 — A human reads the diff (the check no test replaces)

This layer is deliberate, and the system is candid about why.

> The first fully autonomous run **passed every automated check while deleting a
> twelve-line design-rationale comment.** The task said "add a documentation
> block; don't change executable code". The model obeyed the letter of that,
> judged the old comment didn't match, and replaced it. Comments have no
> assertions, so the suite stayed green and the goal came back labelled verified.

That is the case that justifies the whole design:

- **"Green" is a narrow claim, and every verify report says so.** It means: the
  unit suite passes, touched Dart files pass the analyzer, touched JS/HTML/CSS/SQL
  parse. It does **not** mean integration tests ran, Flutter tests ran, docs and
  comments are intact, or the app behaves correctly at runtime.
- The fix was applied where the defect was *written*, not downstream: a
  conventions file that is in the model's context on **every** edit (never remove
  a comment the task didn't name; carry old text through unchanged), and a rule
  for the planner to name what must survive. The identical task re-run went from
  `2 insertions, 12 deletions` to `4 insertions`.
- Both fixes are prompt-level against a small model, and the doc says so: "this
  failure class is not one tests can catch". So the final step is a person
  reading a diff, and **no agent can merge**.

---

## 5. Evidence: it was measured, not assumed

Seven goals were given to the harness on purpose, each scored on correctness,
maintainability, security and efficiency, using `pytest` + branch coverage,
`radon`, `lizard` and `bandit` ([`QUALITY-EVAL.md`](QUALITY-EVAL.md)).

The finding that makes a good demo story is **where the fix went**:

| Defect | Seen | Cause | Fix | After |
|---|---|---|---|---|
| **N+1 query** (one SQL query per row) | Runs 1 and 2 — failed both | Written by the *planner*, copied verbatim | One rule in the planner's instructions: no query inside a loop over rows | Passed 5 of 5 later runs; measured **1 SQL statement for 50 rows** where the old pattern issued 51 |
| **CSV formula injection** (`=HYPERLINK(...)` in an export) | Runs 2 and 4 | Spec silent on it; model didn't think of it | Planner must write a short "code review" section naming each function's query count, bound parameters and output escaping — "a line you cannot honestly write means the code is not ready" | Runs 5–6: cells starting `= + - @` are neutralised and verified |
| **Altered test** | Runs 2 and 4 | Model, content-triggered | A prompt rule — which had **no effect** | Caught reliably by the live test gate (`exit 1`); did not recur in runs 5–7 |

The point to make: this is a loop of *measure → find which layer wrote the
defect → fix that layer → re-run the same spec*. Reviewer agents were
deliberately **not** added, because prompt changes at the source worked.

Security properties confirmed in the generated code: all SQL parameterised;
user text matched literally (`%`, `_`, quotes, and `'; DROP TABLE …;--` match
nothing and leave the table intact); HTML output escaped.

Operational numbers: **2–4¢ per goal**; $0.87 over 911 requests in a recorded
month at ~98% cache hit rate; spend caps of $1/hour, $3/day, $30/month.

---

## 6. What it does *not* do — say this yourself before someone asks

Credibility comes from volunteering these. All are recorded in the repo docs.

- **Green is not a full-suite pass.** Integration, database and live suites do
  not run: they need Docker, and Docker is root-equivalent, which would undo the
  no-sudo isolation. That is a deliberate trade.
- **Flutter tests don't run** — only `flutter analyze`. 115 test files are idle.
- **Nothing executes the app.** Rendering and runtime behaviour are unchecked.
- **Comments, docs and design notes are unguarded by tests.** Only the prompt
  rules and the human diff review protect them.
- **Quality evidence is small.** Seven runs on toy Python/SQLite modules; not
  yet shown on the JavaScript/Dart code where the real work is. These are
  measurements on one problem family, not a guarantee.
- **The retry's repair of the altered-test case is unproven.** Detection is
  proven; recovery from that specific mutation has not been exercised, because
  it didn't recur once the gate was live. (Retry-and-Sentinel recovery from
  *another* real failure was proven end to end.)
- **Loop-back's failing-file parser is a heuristic.** It reads stack frames, so
  it finds the file a failure was raised in, not necessarily the file that broke.
- **Units are serial.** One GPU, so wide goals are slow.
- **Single repo onboarded**, and the board runs in a trust mode where any local
  process on the orchestrator box acts as the board — mitigated by a
  bearer-token-only proxy, but there is no per-person audit trail.

---

## 7. A five-minute demo script

1. **Show the goal.** Open the board and read a goal issue in plain English (a
   small, behaviour-only one works best).
2. **Show the plan.** Glorp's fan-out: an integration branch, one issue per
   unit, and a verify issue **blocked by every unit**. Point out the planner
   wrote no application code.
3. **Watch a unit run.** Tail the job log; it is a real local model on a GPU. Note
   it is the only agent that can edit files, and it can't reach the internet.
4. **Show a gate failing — the strongest moment.** Run the verify against a
   branch from the recorded altered-test case (`agent/goal-purchases-module` on
   `scratch`): it returns `exit 1`, one named failed test, 18 passed. "The model
   changed a test it was told to copy exactly. The suite caught it."
5. **Show the scope statement** at the bottom of a verify report: green means
   *this*, not *that*.
6. **Show the hand-off.** The PR in review, assigned to a human. Open the diff.
   Tell the comment-deletion story: this is why a person reads it.
7. **Show the cost.** `deepseek-usage --balance` and the ledger: pennies per goal.

Safe, read-only commands for the live parts:

```bash
curl -s localhost:4100/_meter/status | jq '{open, windows}'      # spend vs. caps
ssh z440 'systemctl is-active llama litellm'                      # model is up
ssh z440 "sudo -u agent git -C /home/agent/repos/sorbit diff --stat prod-testing...agent/goal-<slug>"
```

Per the repo rules: don't un-assign/re-assign units to wake an agent, don't use
`paperclipai agent wake` to test, and don't touch the Z440's checkouts during a
demo. For a live goal, make sure the board is idle first and use `scratch`
rather than Sorbit.

---

## 8. Likely questions

**"What stops the AI from pushing bad code to production?"** It can't push at
all from the coder's box, and the publisher can only open a PR into
`prod-testing` from a fixed branch pattern, enforced by a hook the service can't
edit. A human merges.

**"What if the model writes a vulnerability?"** Three defences, none sufficient
alone: the code is written in a sandbox with no credentials; the planner's
instructions require parameterised queries, output escaping and a written
self-review; and a human reads the diff. We measured that the first two work on
one problem family and say so.

**"Why not just have another AI review the code?"** We tested the premise. Fixing
the prompt at the layer that wrote each defect removed the efficiency failures
and, with a self-review requirement, the injection gap — so a reviewer agent was
not needed. A cheap deterministic gate beats a second model for "did the tests
still pass".

**"Why a small local model instead of a frontier one?"** Cost, privacy and the
containment story. The trade is that a small model is more likely to be subtly
wrong — which is why the quantisation is not lowered to buy context, and why the
gates and the human exist.

**"What if it loops forever or racks up a bill?"** Three strikes, two loop-back
rounds, a hard unit cap on fan-out, per-run limits, and spend caps enforced by an
external meter that the planner can't alter.

**"Is it proven on real production code?"** The pipeline runs on the real repo;
the quality *measurements* so far are on small Python modules. That gap is
recorded as open.
