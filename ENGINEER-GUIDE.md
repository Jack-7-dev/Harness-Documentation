# Giving work to Glorp: engineer guide

How to reach the orchestrator box over Tailscale, hand a coding goal to
**Glorp**, and pick up the result.

---

## 1. How it works (the 2-minute version)

You write a goal. An AI planner splits it into small edits. A local coding
model makes the edits one at a time on a GPU box, the tests run after each
edit, and when everything is green the goal comes back to you for review.
You don't touch anything in between.

```
 you ──▶ Paperclip web UI (mini PC)
            │  issue assigned to Glorp
            ▼
         GLORP   plans: reads the repo, splits the goal into units,
            │     runs goal-fanout
            ▼
         ZED ×N  one unit at a time: ships each edit to the Z440, where the
            │     local model edits the file and the unit tests run
            ▼
         VERITY  after the last unit: runs the test suite on the combined
            │     branch
            ▼
 you ◀── goal comes back "in review", assigned to a human
```

### The two machines

| Machine | Role | What runs there |
|---|---|---|
| **Mini PC** (`charlie-precision-workstation-t3500`): **the one you Tailscale into** | Orchestrator | Paperclip (the web UI and task board) and the agents' control scripts |
| **Z440** (`jack-1-0`, LAN only: direct cable or wifi) | Executor | The Qwen3-Coder-30B model on the GPU, plus the sandboxed `agent` account that edits code and runs tests |

You log in to the mini PC. From there, `ssh z440` gets you onto the Z440
without a password, which is how you review the finished code (section 5).

### The agents

| Agent | Brain | What it does | When it runs |
|---|---|---|---|
| **Glorp** | DeepSeek V4.1-Flash (cloud API, metered) | **The planner.** Reads the Sorbit repo, decides the smallest set of changes, and writes a plan of *units* (normally one file each, with exact instructions). Then runs `goal-fanout`, which creates the integration branch `agent/goal-<slug>` and one child issue per unit. **It never writes code itself.** | When you assign it an issue, or comment on an issue it holds |
| **Zed** | Qwen3-Coder on the Z440 (local, free) | **The coder.** Takes one unit, ships it to the Z440, where Aider edits the named file(s), commits, and runs the Node unit tests. A unit that changes Dart also runs `flutter analyze` on its touched Dart files before fan-in, and every unit must change each file it names, with real content in it (a file created empty fails the unit). If every applicable check passes, the commit is added to the integration branch. If not, the branch is left alone and Paperclip queues an issue-bound retry automatically; the third failed attempt moves the unit to `blocked`. | Automatically, as each unit issue is assigned to it. **One at a time**, because there's one GPU. |
| **Verity** | none (just runs checks) | **The checker.** Once every unit is done, it runs the Node unit suite against the whole integration branch. If the goal touched Dart files, it also runs `flutter analyze` against only those files. **Green:** marks the goal `in_review` and hands it to a human. **Red:** creates up to 4 scoped fix issues for Zed and re-checks, for at most 2 rounds, then parks the goal as `blocked` for a human. | Automatically, when the last unit finishes. It doesn't need the GPU, so it can run alongside Zed. |
| **Database Operator** | none (runs fixed scripts) | Applies **reviewed, named** live-database runbooks. An issue picks one by name (`Database-Operation: <runbook>` plus `Apply: true`); it never turns issue text into SQL. Not part of the coding flow, so leave it alone unless Charlie asks. | Only when an issue is assigned to it |
| **Sentinel** | DeepSeek Flash (same backend as Glorp) | **The productivity reviewer.** Receives Paperclip's high-churn review issues, examines the source run history and diagnostics, and decides whether to continue, decompose, reroute, close the review, or escalate to the board. It does not edit application code. | Automatically when Paperclip's high-churn watchdog opens a review for Zed. |

### Timing

- A small unit takes **about 20 s to 3 min**, depending on file size.
- A 2-unit goal went from assignment to "in review" in **about 2.5 minutes**.
- Units never run in parallel: a 10-unit goal takes 10× one unit.

---

## 2. One-time setup

### 2a. Get on the tailnet

1. Install Tailscale: <https://tailscale.com/download>. It's available for
   macOS, Windows and Linux. On Linux:
   ```sh
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up
   ```
2. **Ask Charlie to share the mini PC with you.** You'll get an email invite;
   accept it with the account you signed into Tailscale with.
3. Check you can see it:
   ```sh
   tailscale status | grep charlie-precision
   ping charlie-precision-workstation-t3500.tailb31a98.ts.net
   ```

### 2b. Get SSH access

Everyone logs in as the shared **`charlie`** account. **Charlie will give you
the password in person.** Don't write it down anywhere shared: not Slack, not
email, not an issue, not a repo.

Add this to your `~/.ssh/config` so you don't have to type it all each time:

```
Host glorp
    HostName charlie-precision-workstation-t3500.tailb31a98.ts.net
    User charlie
    LocalForward 3100 127.0.0.1:3100
    ServerAliveInterval 30
    ExitOnForwardFailure yes
```

Test it with `ssh glorp`. Type the password when asked; you should land in a
shell on the mini PC. Type `exit` to leave.

> Why a tunnel and not just a URL? Paperclip listens on the mini PC's
> loopback only, so the `LocalForward` line carries it to your machine over
> SSH.

---

## 3. Every time: open the UI

```sh
ssh glorp
```

Enter the password and leave that terminal open. While you're logged in, the
tunnel is up. Browse to **<http://localhost:3100>** and pick the **Sorbit**
company.

That same terminal is also a shell on the mini PC, which you'll use for
reviewing (section 5). Type `exit` when you're done; that closes the UI too.

### House rules for the shared account

`charlie` is a real admin account. Everyone uses it, and it has sudo and
password-free access to the Z440. So:

- **Use the web UI to give work.** Don't run the agent scripts, `goal-fanout`
  or `jobd` by hand.
- **Look, don't change:** `~/harness`, `~/.paperclip`, `/etc/deepseek-meter`,
  systemd services, and anything under `/home/agent` on the Z440.
- **Never** run `npm ci`, `git checkout`, or commit inside the Z440's repos.
  The agents share those checkouts.
- **Don't change the password or add SSH keys.** Ask Charlie.
- Treat the password like a production credential.

---

## 4. Giving Glorp a task

1. In Paperclip, create a **new issue**.
2. Write the goal (see the template below).
3. **Assign it to Glorp.** Assigning is what wakes it. There's no "run"
   button.
4. Walk away. Glorp may reply with a question in the comments if the goal is
   vague. **Answer it in a comment on the same issue**; your comment wakes it
   again.

### How to write a good goal

Glorp is only as good as the goal, and the coding model underneath is small
(3B active parameters). It does exactly what it's told, *literally*. So be
specific.

```
Title:  Fix fingerprint mismatch for questions with trailing punctuation

What:   normalise() in
        app-frontend/sorbit_app/cloud-run/src/.../question-fingerprint.js
        trims whitespace before stripping punctuation, so "What is X ?"
        and "What is X?" fingerprint differently. They should match.

Done when: both strings produce the same fingerprint. Existing tests
        still pass.

Don't touch: the hashing function and the public signature of
        normalise().
```

**Do:**
- **Name files and functions** if you know them. It saves Glorp guessing.
- **Say what "done" looks like**, as behaviour, not "make it better".
- **Say what must survive**, not only what must not break. For example:
  *"add a comment above the existing header. Leave the existing header
  exactly as is."* A real run once replaced 12 lines of design notes because
  the task only said "don't change executable code".
- **Keep it single-PR sized.** Glorp stops and asks you if a goal needs more
  than ~12 units.

**Don't:**
- Ask for whole-repo refactors, renames across the codebase, or "tidy up X".
- Assume a Flutter analyzer pass is the same as a Flutter test pass. Flutter is
  installed on the Z440, and the harness runs `flutter analyze` on touched Dart
  files with informational lints non-fatal, but it does not run the Flutter
  widget/unit tests yet (see limits below).
- Write `Tests:` or `Base:` lines. Those are filled in automatically.
- Paste secrets, credentials or customer data into an issue. Glorp's text
  goes to DeepSeek's API.

---

## 5. What you'll see, and what to do with it

| You see | Meaning | You do |
|---|---|---|
| Glorp comments a summary of units | Plan made, units queued | Nothing, just wait |
| Child issues appear, assigned to **Zed** | Units running one by one | Nothing. Click in to watch progress. |
| A **Verify** child assigned to **Verity** | Waiting for the last unit | Nothing |
| Goal goes **`in_review`**, assigned to you/the board | ✅ All units landed, suite green. The handback comment links a **pull request** into `prod-testing` | **Review the PR** in GitHub (below) and merge it yourself |
| A unit is assigned to **Sentinel** | It failed 3 attempts; Sentinel is diagnosing it and rewriting the task | Nothing. It goes back to Zed for 3 more attempts |
| Goal or a unit is **`blocked`** | A unit failed 3 times after Sentinel's rewrite (or Sentinel judged it unfixable), or verify stayed red after 2 fix rounds | Read Sentinel's diagnosis and the failure comments. Re-scope the goal, or fix it by hand. |
| Glorp asked a question | Goal too vague to split safely | Answer in a comment |

### Reviewing: this step matters

"Green" means **the `cloud-run` unit tests pass, every touched Dart file
passes `flutter analyze`, and touched JS/HTML/CSS/SQL files pass a syntax-only
check**, and nothing more:

- **Integration, Postgres and live suites do not run.** The sandbox has no
  docker, by design.
- **Flutter is installed, but Flutter tests do not run.** The final gate runs
  `flutter analyze` only on Dart files changed between `prod-testing` and the
  integration branch. This catches syntax, type and API errors without making
  the goal fail on informational style lints or the repo's pre-existing
  analyzer findings. Informational findings remain visible in the log. It does
  not run `flutter test`, render widgets, or prove runtime behaviour.
- **Tests don't read comments or docs.** A deleted comment or a
  wrong-but-passing change sails through.

### Glorp's read-only infrastructure access

Glorp has no general Internet route. `infra-reader` is the credential boundary
for source freshness and production diagnostics. It holds separate read-only
GitHub App, Neon and GCP identities and exposes only three bounded operations:

```sh
infra-read sync
infra-read neon 'SELECT table_name FROM information_schema.tables WHERE table_schema = '\''public'\'''
infra-read gcp logs --filter 'resource.type="cloud_run_revision"' --limit 50
```

The GitHub operation refreshes `/var/lib/infra-reader/Sorbit`, which is mounted
read-only for Glorp. Neon queries are single-statement `SELECT`/`WITH` queries
inside a forced read-only transaction with row, time and output caps. GCP uses
fixed viewer commands; arbitrary `gcloud`, IAM, deployment and secret-value
access are not available. Credentials never enter Glorp's environment. Audit
records are available with `journalctl -u infra-reader.service`.

So **always read the diff.** On a green verify the branch `agent/goal-<slug>`
is pushed to GitHub and a pull request into `prod-testing` is opened; the
handback comment on the goal links it. Review and merge it there like any PR.
Nothing merges automatically. If the comment says **PR not opened**, it gives
the reason, and the branch is still on the Z440. Read it there from your
`ssh glorp` shell:

```sh
# list goal branches
ssh z440 'sudo -u agent git -C /home/agent/repos/sorbit branch --list "agent/goal-*"'

# what changed (summary, then each commit)
B=agent/goal-<slug>
ssh z440 "sudo -u agent git -C /home/agent/repos/sorbit diff --stat prod-testing...$B"
ssh z440 "sudo -u agent git -C /home/agent/repos/sorbit log -p prod-testing..$B" | less
```

These commands only read. The push is done by the harness's publisher
(`infra-reader`), never by an agent directly: it only ever writes
`agent/goal-*` branches, never force-pushes, only opens PRs into
`prod-testing`, and refuses any branch that touches `.github/`. A pre-push
guard also refuses any push while the publisher's copy of `prod`,
`pre-testing`, `prod-testing` or `master` differs from GitHub. Merging is
always a human's job.

---

## 6. Limits to know

| Limit | Why |
|---|---|
| **Sorbit repo only** for now | It's the only repo onboarded on the Z440 |
| **One unit at a time** | One GPU. Big goals queue, they don't parallelise. |
| **Node unit tests + scoped Dart analysis only** | Flutter widget/unit tests, integration, Postgres and live suites remain outside the gate. See "Reviewing" above. |
| **Glorp spend caps: $1/hour, $3/day, $30/month** | Typical goal costs ~2–4¢. If a cap is hit, Glorp is paused until the window rolls over. Tell Charlie if that happens. |
| **Very large files are slow** | A 222 KB file works but takes ~3 min per edit. Smaller, targeted files are faster and more accurate. |

---

## 7. Troubleshooting

| Problem | Fix |
|---|---|
| `ssh: Could not resolve hostname` | Tailscale isn't connected. Run `tailscale status`; if the mini PC isn't listed, accept the share invite. |
| `Permission denied` | Wrong password. Check it with Charlie, in person. |
| `ssh z440` asks for a password | You ran it on your own machine. Run it from inside your `ssh glorp` session. |
| `bind: Address already in use` | Something else on your machine uses port 3100. Change `LocalForward 3100` to `LocalForward 3200 127.0.0.1:3100` and browse to `localhost:3200`. |
| Assigned to Glorp, nothing happens | Assignment only wakes Glorp when the assignee **changes**. Unassign, save, then assign again. This is safe for a goal you created, but **never do it to a unit** (a child issue assigned to Zed): Paperclip hands an unassigned unit to Glorp. Ask Charlie instead. |
| Glorp paused / "cap" in the run log | A spend cap was hit. It resumes by itself when the window rolls over. |
| Goal stuck on a unit for > 1 hour | Jobs time out at 1 hour. Check the unit's comments, then tell Charlie. |

---

## Appendix: for Charlie (admin)

**Sharing the machine.** The tailnet belongs to `jackbower835@gmail.com`. In
the Tailscale admin console, open *Machines*, then `charlie-precision-workstation-t3500`,
then **Share**, and invite each engineer by email. That gives them this one
machine without joining them to the whole tailnet.

**Access is the shared `charlie` password,** given out in person. Password
SSH login is already on (`PasswordAuthentication yes`). Keep in mind what the
password unlocks:
- passwordless sudo here;
- `ssh z440` as `jack`, who also has passwordless sudo on the Z440;
- the DeepSeek key file, through sudo.

**Revoking access** means changing the password (`passwd`) and telling
everyone else the new one. If an engineer leaves, do it that day.

**If you later want per-person access,** give each engineer a tunnel-only key
instead. Add a line to `~/.ssh/authorized_keys`:

```
restrict,port-forwarding,permitopen="127.0.0.1:3100",command="/bin/false" ssh-ed25519 AAAA... engineer-name
```

That lets them reach the UI but not a shell. You revoke one person by
deleting their line. Reviewing diffs would then go through you.

**Also worth knowing:** Paperclip is in `local_trusted` mode, so everyone
acts as the same "board" user and there's no per-person audit trail in the
UI either. Switching to `authenticated` + `private` with `bind: tailnet`
gives everyone their own login. Re-check the `paperclip-gate` / glorp
isolation from `MINIPC.md` after that switch.
