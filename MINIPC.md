# Mini PC (orchestrator) — build state

The orchestrator box. The Z440 is in `Z440.md`; the overview is in `README.md`.
Written 2026-09-04; state re-checked against the live box 2026-09-18.

| Thing | Value |
|---|---|
| Host / IP | `charlie-Precision-WorkStation-T3500` / `192.168.1.121`; tailnet `100.114.50.113` |
| OS | Linux Mint 22.1 (Xia), Ubuntu 24.04 base |
| CPU / RAM | 4 cores, 7 GB |
| Disk | 388 GB free on `/` (2026-09-18) |
| sudo | **passwordless** for `charlie` (was password-gated on 2026-09-04, when the toolchain below was installed user-local) |
| Role | orchestrator: Paperclip control plane + planner + dispatcher, and the box engineers reach over Tailscale (`ENGINEER-GUIDE.md`) |

## What is on this box

**Toolchain** — all user-local, no root:
- Node **24.20.0** via nvm, set as `default` (Paperclip needs ≥24.11; the box had 18)
- pnpm **11.25.0** via corepack
- `jq` 1.7.1 and `uv` 0.12.9 in `~/.local/bin`

**Fixed a live `~/.bashrc` bug.** Line 124 was
`export PATH="$HOME/.local/bin/claude"` — an assignment, not a prepend, so every
interactive shell ended up with *only* that one directory on `PATH`. Now
`…/claude:$PATH`. Backup at `~/.bashrc.bak-20260904`.

**Network link** — `~/.config/systemd/user/z440-tunnel.service`
Runs `ssh -NT -i ~/.ssh/z440_key -L 127.0.0.1:8080:localhost:8080 jack@192.168.1.199`
with `ExitOnForwardFailure`, `ServerAliveInterval=30`, `Restart=always`.
Linger was already on, so it starts at boot without a login session.
Verified: killed the ssh process, systemd rebuilt it in <8s.

```bash
systemctl --user status z440-tunnel     # state
journalctl --user -u z440-tunnel -f     # logs
```

**Verified through the tunnel** (acceptance rung 1, from this box):
- `/health` → ok, `/v1/models` → the model
- a real completion in 3.5 s (JIT already paid)
- **tool calling** → `finish_reason: tool_calls` with a well-formed call

**SSH keys**
- `~/.ssh/z440_key` — existing, full shell as `jack`. Host alias `z440`.
- `~/.ssh/z440_jobd` — dedicated to the job channel, pinned to the `jobd`
  forced command. Host alias `z440-job`. Kept separate from `z440_key` so
  the forced command can't be sidestepped.
- `~/.ssh/z440_verify` — pinned to the `verifyd` forced command. Host alias
  `z440-verify`.

**Paperclip** — managed install, `paperclipai` **2026.831.1** in `~/.paperclip`,
server on `127.0.0.1:3100` (loopback), embedded Postgres on `:54329`.
Installed via the CLI directly: the published `install.sh` passes `--no-prompt`,
which this CLI version rejects (upstream mismatch).
Runs as `paperclipai.service` (systemd --user, enabled, linger on) — **not** as a
foreground process, after a stray `timeout` killed the onboarding run once.
One drop-in, `~/.config/systemd/user/paperclipai.service.d/10-heartbeat-interval.conf`,
sets `HEARTBEAT_SCHEDULER_INTERVAL_MS=10000` (2026-09-28): the monitor tick that
fires Zed's unit retries rides the heartbeat scheduler, and its 30s default was
the whole retry latency. 10000 is the floor the server enforces; anything lower
is silently raised. A drop-in rather than an edit, so `paperclipai service
install` regenerating the unit keeps it. The banner confirms it on boot:
`Heartbeat  enabled (10000ms)`.
Company **Sorbit** `ddeaf594…`. Sorbit is the real product at `~/Sorbit` /
`github.com/Sorbit-ltd/Sorbit`.

Agents on the board (2026-09-18):

| Agent | id | adapter | runs | notes |
|---|---|---|---|---|
| **Glorp** | `c0e87781…` | `claude_local` | Claude Code as user `glorp`, via `/usr/local/lib/glorp/launch/claude`, on `deepseek-flash` | planner; timer off, wakes on assignment/comment; 1800 s / 200 turns |
| **Sentinel** | `0ed506f6…` | `claude_local` | same launcher and model as Glorp | reviews Paperclip's high-churn watchdog issues |
| **Zed** | `03887377…` | `process` | `~/harness/bin/z440-exec` | coder bridge to `jobd`; assignment only; 5400 s |
| **Verity** | `dd57e84b…` | `process` | `~/harness/bin/z440-verify` | end-of-goal verify + loop-back; assignment only |
| **Database Operator** | `1a33e10a…` | `process` | `~/harness/db-operator/db_operator.py` | non-LLM; runs reviewed, named live-DB runbooks only. See `~/harness/db-operator/README.md` |

**Claude quota guard** — `~/harness/bin/claude-quota-guard`
+ `claude-quota-guard.timer` (every 5 min).
Reads `api.anthropic.com/api/oauth/usage` with the local OAuth token and pauses
every `claude_local` agent when a window goes over threshold; resumes only the
agents it paused itself. Default **weekly 50%**, **session 85%**.
Log: `~/harness/logs/quota-guard.log`, state: `~/harness/state/quota-guard.json`.

Since 2026-09-11 Glorp and Sentinel run on DeepSeek, not the subscription. The
guard treats them as "metered": it pauses them while the meter's
`/_meter/status` says a spend cap is closed. With no subscription agent left, it
no longer calls the Claude usage endpoint. See [Glorp on DeepSeek](#glorp-on-deepseek).

**System services added for DeepSeek and Glorp's isolation** (root, 2026-09-11 onward):
`deepseek-meter` (:4100, user `dsmeter`), `paperclip-gate` (:3101),
`glorp-firewall`, the `home-glorp-Sorbit.mount` read-only bind, and
`infra-reader` + `infra-reader-sync.timer` (Glorp's read-only GitHub/Neon/GCP
gateway, described in `ENGINEER-GUIDE.md`). Since 2026-09-21 infra-reader also
has one write: `/v1/github/publish`, called only by Verity with a separate
publish key, pushing `agent/goal-*` branches through a pre-push guard and
opening PRs into `prod-testing` (see `FANOUT.md`). `infra-reader-sync` also
triggers `z440-sorbit-push` (`OnSuccess=`), which fast-forwards the Z440's
Sorbit clone from the mirror. Also `tailscaled`.

**Claude Code:** 2.1.276 for `charlie`; Glorp's copy in `/opt/glorp` is pinned at
2.1.268 and changes only with `sudo glorp-sync-tools`.

> See `Z440.md` for the executor half of this harness.

> **Why this exists — Paperclip's budgets cannot cap subscription work.**
> `budgetMonthlyCents` is the only hard-stop, and
> `heartbeat.ts::normalizeBilledCostCents()` returns **0** for
> `billingType === "subscription_included"`. A `claude_local` agent on a Pro
> subscription therefore never accrues spend, so no cents budget will ever
> fire. `quota-windows.ts` is 64 lines of read-only display with no thresholds,
> and heartbeat.ts only reacts to `provider_quota` *after* a run has already
> failed. Hence an external guard.

**Bridge** — `~/harness/bin/z440-exec`, Python 3, no dependencies.
One heartbeat = claim an assigned issue → `ssh z440-job` → comment the result →
set status. Includes the plan's 3-strike rule (3 failures → `blocked`); a failed
attempt schedules an issue-bound retry that carries feedback from the failure.
`--selftest` checks API + SSH; `--dry-run` prints the job JSON without dispatching.
Its siblings: `z440-verify` (Verity's bridge), `goal-fanout` (Glorp's plan →
units), `glorp-brain` (switch Glorp between DeepSeek and Claude).

## Corrections to the original plan

1. **No custom adapter is needed for Aider.** The plan says the process adapter
   has no generic run-any-command mode and budgets ~100 lines of custom
   `ServerAdapterModule`. `server/src/adapters/process/execute.ts` takes
   `command`, `args`, `cwd`, `env`, `timeoutSec`, `graceSec` and spawns anything.
   The real constraint is different: **it does not pipe the task to stdin.** It
   injects `PAPERCLIP_API_URL` / `PAPERCLIP_API_KEY` / `PAPERCLIP_AGENT_ID` /
   `PAPERCLIP_RUN_ID` and the agent fetches its own work. Hence the bridge script.
2. **`hermes_gateway` and `hermes_local` are built in** — confirmed in
   `BUILTIN_ADAPTER_TYPES`, supplied by `@paperclipai/hermes-paperclip-adapter`.
   The plan was right about this.
3. **The onboarding wizard offers no generic adapter.** "Connect a model" lists
   only vendor CLIs (Claude Code, Codex, Cursor, Gemini, Grok, Kimi, Hermes,
   OpenCode, Pi). Pick **Claude Code** — it's the only one already installed
   here, and it matches the plan's recommended cloud planner. The Z440 executor
   is added afterwards as a `process` agent.

## Build checklist (all done)

- [x] `paperclipai service install` — enabled, linger already on
- [x] Executor agent created: **Zed**, adapter `process`, command
      `/home/charlie/harness/bin/z440-exec`, env `Z440_DEFAULT_REPO=sorbit`,
      `timeoutSec` 5400 (above jobd's 3600 s aider timeout)
- [x] End-to-end run — SOR-3, 2026-09-07 (`Z440.md`, rung 7)
- [x] Z440 side built: `agent` user, `/etc/llama` key, `ufw`, loopback bind,
      `qwen3-coder` alias, and the `jobd` key installed as a forced command
      (now also `no-port-forwarding,restrict`)

---

## Glorp on DeepSeek

Moving Glorp's brain from Claude to DeepSeek: the plan, how it was built, and
how it is running now.

Status: **installed and live since 2026-09-11** (meter up since 13:53 UTC).
Written 2026-09-11 as a plan; the steps below are kept as the record of how
it was built. The design as planned:
- Glorp uses `deepseek-flash` (V4.1-Flash).
- A metering proxy is added, and it holds the key.
- Glorp moves to its own `glorp` user (step 1b), so `charlie` can keep
  passwordless sudo.

Where things stand (checked 2026-09-18):
- **Live:** `deepseek-meter`, `paperclip-gate`, `glorp-firewall` and the
  read-only Sorbit mount are running. Glorp runs through the launcher on
  `deepseek-flash`, with the planned run limits (1800 s, 200 turns, 2
  concurrent). **Sentinel** was added later on the same launcher and model.
- **Caps as planned:** $1/hour, $3/day, $30/month.
- **Measured spend, September to date:** $0.87 over 911 requests. Cache hit rate
  is ~98% (63.5M tokens from cache vs 1.35M uncached), well above the level
  that would reopen Decision 1.
- **Not recorded:** step 7 said to write the switch and cost per run into
  `FANOUT.md`. That never happened; the figures above stand in for it.
  Whether step 6's cap-refusal drill was run is not documented either.
- Since then Glorp also reads GitHub, Neon and GCP through `infra-reader`, a
  read-only gateway (see `ENGINEER-GUIDE.md`). So the "no internet" isolation
  below now has one deliberate, credential-isolated exception.

Glorp is the orchestrator: it takes a goal, writes the plan, and fans it out
to Zed and Verity on the Z440. Today it runs on the Claude subscription. This
plan moves it to the DeepSeek API, paid per token, with a key you enter by
hand. Zed, Verity, jobd, the Z440 and the local Qwen model are not touched.

### Shape

```
Paperclip (charlie) ── claude_local, engine=cli, command=/usr/local/lib/glorp/launch/claude
   │  launcher: env allowlist, stages prompt bundle, sudo -u glorp (that script only)
   ▼
Glorp's Claude Code  (user glorp: no sudo, no_new_privs, uid firewall)
   │  ANTHROPIC_BASE_URL=http://127.0.0.1:4100      PAPERCLIP_API_URL → :3101
   ▼                                                 ▼
deepseek-meter  (user dsmeter, holds the key)       paperclip-gate → Paperclip :3100
   │  counts every token, prices it,                 (forwards bearer-token calls only)
   │  refuses once a cap is hit
   ▼
https://api.deepseek.com/anthropic   (model forced to deepseek-flash)
```

### Measured 2026-09-11: DeepSeek's cache, and Claude Code running on it

The question was whether DeepSeek caches what Claude Code re-sends every turn,
whether it bills the cached part cheaply, and whether that holds at 1M-token
scale. If not, the plan would switch to a harness that calls the API directly.
**All three hold.** Total spend on the tests: $0.16, all off-peak.

**1. Raw API, large prefix.** An 840k-token block was sent once, then re-sent
three times with a different question on the end:

| call | uncached | from cache |
|---|---|---|
| 0 (first send) | 786,685 | 53,632 (left over from an earlier probe) |
| 1 (+8 s) | 147,581 | 692,736 (cache still building) |
| 2 | 253 | **840,064** |
| 3 | 253 | **840,064** |

- **Billing:** the balance fell **$0.14**. Predicted: $0.1475 with the cache
  discount, $0.50 without it. Cached input is billed at the hit price.
- **Balance lag:** the balance updates 1–3 minutes after the calls.
- **Build lag:** the cache builds in the background. A very large *new* chunk
  re-sent within about 8 s only partly hits. Claude Code turns add small
  deltas, so this doesn't bite in practice (next table).

**2. Claude Code CLI 2.1.268 on DeepSeek.** The setup:
- A fresh `CLAUDE_CONFIG_DIR`, so there was no Claude login to fall back on.
- `deepseek-flash`, with read-only tools.
- The task: read five files of Paperclip's own source and explain its cost
  accounting.

The results:
- 15 turns and 13 API calls, done in 40 s. The answer was correct, with
  accurate line citations.

| | tokens |
|---|---|
| input from cache | 500,736 (**90.4%**) |
| input uncached | 53,027 |
| output | 4,716 |

- **Cache hit rate per call:** 0% on the first call, 52% on the second, 83%
  on the third, then **95–99.6%** on every later call. The uncached part of
  each call was just the file it had read.
- **Billing:** the balance fell $0.01, against $0.0123 computed at off-peak
  rates. At peak it would be ~$0.025. Without the cache it would be $0.08.
- **Claude Code's own `total_cost_usd` said $0.63**, pricing the run as
  Claude, about 50× too high. That is what Paperclip would record. Confirmed:
  the meter's ledger has to be the source of truth.
- **Warning in stderr:** Claude Code logs `unrecognized_model` for
  `deepseek-flash`. It is harmless and the run succeeded, but it's the
  pricing-table miss behind the wrong cost.

**The "1.2M tokens per run" figure is a total, not a cache size.** It adds up
the context re-sent by *every* call in a run. A single call's context was
16–54k here, and about 60k in Glorp's Claude runs. The cache only ever needs
to hold one call's prefix, and that is proven up to 840k. Flash's context
tops out at 1M anyway.

**Verdict: keep Claude Code as the harness.** Its re-sending is cached and
billed at the hit price, so a direct-API harness would save only the per-turn
uncached deltas. That's not worth rebuilding and re-proving Glorp's
tooling.

### Decision 1: keep Claude Code as the harness

Glorp is not a single prompt. It is an agent:
- It reads the target repo to scope units.
- It talks to Paperclip through its skills.
- It writes `plan-*.md` and runs `goal-fanout`.

It has been tuned and tested against Claude Code's tool loop, file tools and
permission model. Routing the DeepSeek API "directly" means writing and
maintaining our own agent loop, tool set, session handling and Paperclip
integration, then re-proving Glorp's scope discipline on top of it.

The real benefit of a direct route is **seeing and controlling every request**.
The meter below gives exactly that, with the harness left alone.

Claude Code's overhead is not the cost driver. It re-sends the whole context
each turn: about 1.2M cached input tokens per Glorp run, measured. At flash's
cache-hit price that comes to under a cent.

The one thing that would change this decision is a **low cache-hit rate** on
DeepSeek's side. Then that re-sending gets billed at the cache-miss price,
50× more. The meter reports hit rate from the first run, so this gets decided
on data. If it comes to that, the next step is `opencode-local` (DeepSeek
native, already in Paperclip), not a harness of our own.

### Decision 2: `deepseek-flash` for Glorp

- **What it is:** V4.1-Flash. 1M-token context, tool calls, thinking mode, and
  vision, which v4-pro lacks.
- **Price per M tokens:** cache hit $0.003 / $0.006, miss $0.15 / $0.30, output
  $0.60 / $1.20, off-peak / peak.
- **Peak hours:** 01:00–04:00 and 06:00–10:00 UTC on weekdays.
- **Quality:** the claim that it's close to Opus 5 is untested for this job.
  The test goal (step 6) is where it has to show it.
- **Fallback:** if plans come out weak, `deepseek-v4-pro` is a one-line change
  in the meter's config.
- **A pitfall that goes away:** DeepSeek silently maps unknown model names to
  flash, which was a risk when pro was the target. The meter also rewrites the
  `model` field on every request, so Glorp can't switch models itself.

#### What a Glorp run should cost

From Glorp's two runs that recorded usage (09-04): **58k fresh input, 1.1–1.3M
cached input, 6–15k output** per run.

| per run | flash off-peak | flash peak | v4-pro peak | Claude (API list, recorded) |
|---|---|---|---|---|
| cache working | ~$0.02 | ~$0.04 | ~$0.19 | $0.78–0.98 |
| cache missing entirely | ~$0.22 | ~$0.43 | ~$1.83 | — |

**Runaway worst case today:** Glorp has `timeoutSec: 0` (no limit) and
`maxTurnsPerRun: 1000`. One looping run re-reading a 500k context for 1000
turns would cost about **$150** at flash peak with the cache missing. Hence
the layers below.

### Decision 3: spend control, four layers

| layer | catches | acts |
|---|---|---|
| **Run limits** on Glorp: `timeoutSec` 1800, `maxTurnsPerRun` 200, `maxConcurrentRuns` 2 | one run looping | Paperclip kills the run |
| **Meter caps**: hourly / daily / monthly $ | runaway across runs, and within a run mid-flight | refuses the next request, so the run fails |
| **Guard**: reads the meter's status on its existing 5.5-min timer | Glorp being woken again and again into a closed cap | pauses Glorp in Paperclip, resumes when the window rolls over |
| **Prepaid balance** at DeepSeek | everything above failing | DeepSeek refuses calls |

Suggested starting caps: **$1/hour, $3/day, $30/month**, about 75 flash runs a
day. They live in the meter's config, which Glorp can't edit (see isolation).

#### `deepseek-meter`: what it does
A small Python stdlib-only service on the mini PC, listening on
`127.0.0.1:4100`.
- **Forwarding.** Passes `/v1/messages` through to DeepSeek, streaming
  included. It swaps in the real key and forces `model`.
- **Metering.** Reads usage from each response: `input_tokens` and
  `cache_read_input_tokens` from `message_start`, `output_tokens` from
  `message_delta`, and the plain body for non-streamed calls. Each request is
  priced by the peak/off-peak rate at its UTC time. If DeepSeek doesn't report
  cache reads, all input is charged as a miss, so the meter errs high.
- **Ledger.** One line per request to `ledger.jsonl`: time, model, tokens by
  kind, and $.
- **Enforcement.** Before forwarding, it checks the rolling hourly, daily and
  monthly spend against the caps. Over a cap it returns a **non-retryable**
  error, so Claude Code fails the run at once instead of retrying.
  **Confirmed: 403 `permission_error`.** Claude Code 2.1.268 exited in about
  1 s with the cap message, made one request and no retries. The windows
  are a rolling hour, a rolling 24 h and the calendar month (UTC).
- **Run attribution.** The launcher adds `X-Paperclip-Run-Id` through
  `ANTHROPIC_CUSTOM_HEADERS`, so each ledger line names its Paperclip run.
  The meter strips it before forwarding.
- **Fails closed.** A model with no price table is refused at startup, not
  metered at $0. So switching to v4-pro means adding its prices first.
- **Reconciliation.** It polls DeepSeek's `GET /user/balance` hourly. It
  records the drift between "the balance went down by $X" and "the meter
  counted $Y", and the status view shows it. The ledger has to agree with the
  bill, or it isn't trusted.
- **Status view.** `GET /_meter/status` (JSON), used by the guard. A
  `deepseek-usage` command prints spend per hour, day and month, tokens, cache
  hit rate, cost per Glorp run, the balance and the drift.

#### Isolation: the meter and Glorp each get their own user
Glorp has shell access and skipped permissions. Running as `charlie`, with
`charlie` holding passwordless sudo, one `sudo cat` would read the key or
raise a cap. So both sides move out of `charlie`:
- The meter runs as a `dsmeter` system user.
- The key goes in `/etc/deepseek-meter/key` (0600, dsmeter).
- The caps go in `/etc/deepseek-meter/config.json` (root-owned, dsmeter-readable).
- The ledger lives in `/var/lib/deepseek-meter`, world-readable.
- **Glorp runs as a `glorp` system user** (step 1b). It has no sudo, and
  `no_new_privs` means nothing setuid works under it. Its home is
  `/home/glorp`, and it has its own Claude config dir, so it can't see your
  Claude login.
- The meter's client key is readable by the `glorp` group only. Glorp's
  launcher supplies it, and Paperclip's config holds a placeholder, so
  nothing else on the box can spend through the meter.

**What a second user could still reach, found while building step 1b:**
- **Paperclip's API.** It runs `local_trusted`, so any loopback request
  *without* a bearer token is treated as the instance-admin board
  (`server/src/middleware/auth.ts`, `actorMiddleware`). The board can rewrite
  a `process` agent's command, and that command runs as `charlie`.
  - Fix: `paperclip-gate` on :3101 forwards only requests that carry a bearer
    token.
  - Paperclip then authenticates that token as an agent, whose permissions
    forbid adapter-config changes (`assertNoAgentAdapterConfigMutation`), or
    rejects it. A bogus token gets a 401 from Paperclip, not the board.
    Tested.
- **Paperclip's Postgres.** The password is the fixed `paperclip` and the
  role is a superuser. The socket was `/tmp/.s.PGSQL.54329` at mode 0777.
  Fix: the socket becomes 0700, via `unix_socket_permissions`.
- **Other loopback services:** OpenClaw's gateway (:18789, :18792), system
  Postgres, containerd, the ssh tunnel on :8080 and CUPS.
  - Fix: a uid firewall (`glorp-firewall`, iptables owner match). glorp may
    connect only to the meter (:4100), the gate (:3101) and the Z440's ssh.
    Everything else is refused, including the internet and IPv6.
    *(Later added: `infra-reader` on :4101.)*

What crosses from `charlie` to `glorp`:
- Paperclip starts Glorp through `/usr/local/lib/glorp/launch/claude`.
- The launcher filters env to an allowlist and points `PAPERCLIP_API_URL`
  at the gate.
- It copies the prompt bundle and MCP config (which live in `~/.paperclip`,
  mode 0700) into `/var/lib/glorp/stage`.
- It then runs `sudo -u glorp claude-inner`. The sudoers rule allows that
  one script and nothing else.
- `claude-inner` pins HOME, cwd (`/home/glorp/work`), PATH and the base URL.
- It is a child subreaper with pdeathsig, so when Paperclip kills the run,
  every process under it dies too. `charlie` can't signal glorp's processes
  directly.

What glorp can use:
- A read-only bind mount of `~/Sorbit` at `/home/glorp/Sorbit`.
- A copy of the jobd forced-command key, which is the same access Glorp had
  already.
- Root-owned copies of `claude` (pinned at 2.1.268, never auto-updated),
  `node`, `jq`, the Paperclip CLI and `goal-fanout` in `/opt/glorp`.
  Refresh them with `sudo glorp-sync-tools`. The Claude pin is a version in the
  script (`GLORP_CLAUDE_VERSION`, default 2.1.268). Until 2026-09-18 the script
  copied whatever `charlie` had installed, so a routine `goal-fanout` sync
  silently moved Glorp to 2.1.276; it was restored before Glorp ran. Move Glorp
  deliberately with `sudo GLORP_CLAUDE_VERSION=x.y.z glorp-sync-tools`.

This replaces the Paperclip-secret idea in the first draft: the key now lives
in exactly one place, which Glorp can't reach.

### Steps

0. **Fix the quota guard. Done.**
   - `pct()` no longer reads utilization `1.0` (1%) as 100%.
   - A failed reading is recorded with its time, and the log gives the age
     of the last good reading.
1. **You: DeepSeek account.**
   - Top up a small prepaid balance, say $10–20. That's the last-resort cap.
   - Create an API key.

   **1b. Me: move Glorp to its own user. Built; you install it.** See
   "Isolation". Sources are in `~/harness/glorp-user/`.
2. **Me: build the meter. Built and tested; you install it.** Sources are in
   `~/harness/deepseek-meter/`.
   - `test/run_tests.py` runs a private meter against `fake_upstream.py`.
   - It covers streaming, non-streaming, pricing, key swap, model forcing,
     header stripping, balance reconciliation, cap refusal, and the real
     Claude Code CLI both over the cap and through a tool round-trip.
   - All pass, and nothing was spent.

   **Install (you, root):**
   ```sh
   sudo ~/harness/install-deepseek-glorp.sh     # users, meter, gate, firewall, sudoers, /opt/glorp
   sudo ~/harness/glorp-user/test-glorp-e2e.sh  # whole path vs the fake upstream; restores config after
   ```
   The e2e test runs the launcher as Paperclip would. Glorp's own Claude Code
   then runs `glorp-probe` from inside its tool loop, which checks that:
   - it runs as glorp, and sudo, the key, `charlie`'s home, Paperclip :3100,
     Postgres, OpenClaw and the internet are all closed;
   - the meter, the gate, jobd and read-only Sorbit are all open.

   It also checks that a SIGTERM or a SIGKILL from Paperclip leaves no glorp
   process behind.
3. **You: enter the key.** In your own terminal (not via `!` here), so it never
   touches this chat or a shell history:
   ```sh
   read -rsp 'DeepSeek key: ' k && printf '%s' "$k" | sudo tee /etc/deepseek-meter/key >/dev/null; unset k
   sudo chown dsmeter: /etc/deepseek-meter/key && sudo chmod 600 /etc/deepseek-meter/key
   sudo systemctl restart deepseek-meter
   ```
4. **Me: repoint Glorp.** `~/harness/bin/glorp-brain deepseek`.
   - The snapshot is already saved at
     `~/harness/state/glorp-config.claude.json`.
   - `engine: cli`. The ACP lane Glorp used before spawns its own agent
     binary, so it can't take the launcher. The CLI lane is plain Claude
     Code, the path proven on DeepSeek.
   - `command` becomes the launcher, `model` becomes `deepseek-flash`, and
     the env is as planned. `ANTHROPIC_API_KEY` is a placeholder that
     `claude-inner` replaces.
   - Run limits: `timeoutSec` 1800, `maxTurnsPerRun` 200,
     `maxConcurrentRuns` 2.
5. **Me: update the guard. Done.**
   - `claude_local` agents on the launcher, or with a non-Anthropic base URL,
     are "metered".
   - They are paused while `/_meter/status` says closed and resumed when it
     reopens. An unreachable meter changes nothing.
   - The Claude usage endpoint is only called while a subscription agent is
     left.
6. **Test with one small goal**, the same size as SOR-10:
   - **Routing:** the run log names `deepseek-flash`, and the Claude session %
     stays flat.
   - **Metering:** the meter's $ for the run matches the drop in DeepSeek's
     balance (`deepseek-usage --balance`, a few minutes after the run). Note
     the cache hit rate.
   - **Quality:** same bar as SOR-10. A scoped unit, a correct `Files:` list,
     do-not-change rules stated, the fan-out lands, and Verity goes green.
   - **Cap refusal:** set a $0.01 hourly cap and wake Glorp. The run fails fast
     with the cap message, the guard pauses Glorp, and there's no retry storm.
     Then restore the cap.
     - To change caps, edit `/etc/deepseek-meter/config.json`, then
       `sudo systemctl reload deepseek-meter`.
7. **Decide.**
   - **Good:** record the switch and measured cost per run in FANOUT.md.
   - **Weak plans:** try v4-pro through the meter config first (add its
     prices), and check whether the failure is a prompt fix.
   - **Low cache hit rate:** evaluate `opencode-local`.

### Rollback (a minute)

`~/harness/bin/glorp-brain claude` restores `glorp-config.claude.json`. Glorp
is then back on the subscription, as `charlie`, under the guard as before.
Its instructions work unchanged under either user. `systemctl stop
deepseek-meter` cuts DeepSeek off entirely; the key file can stay or be
shredded. The glorp user, gate and firewall can stay: they do nothing while
Glorp isn't pointed at them.

### Things to be aware of

- **Where the data goes.** Goal text, Glorp's plans and any repo files Glorp
  reads go to DeepSeek instead of Anthropic: a different provider, under a
  different data policy. Code execution and generation stay on the Z440.
- **Ignored fields.** DeepSeek ignores `cache_control`, thinking
  `budget_tokens`, `mcp_servers` and most `anthropic-beta` headers. None of
  these is something Glorp's flow depends on. Its caching is automatic, and
  the meter measures whether it's actually hitting.
- **Paperclip's `costUsd` will be wrong.** Claude Code prices runs from its own
  Claude table. The meter's ledger is the source of truth for spend.
  Paperclip's token counts are still right and can cross-check it.
- **Claude Code updates.** The Claude Code CLI on the mini PC (2.1.268) may
  start assuming a real Anthropic backend. Glorp's copy in `/opt/glorp` is
  pinned and only changes when you run `sudo glorp-sync-tools`. If Glorp's
  runs break after a sync, suspect that first.
- **ACP lane to CLI lane.** Glorp's earlier runs went through Paperclip's ACP
  engine. It now uses the CLI lane: the same Claude Code, but its first
  session starts fresh. The step-6 goal is the check that nothing in its
  tuning depended on ACP.
- **The Z440's address is in two places:** `glorp-firewall` and glorp's
  `~/.ssh/config`. It changed once already.
- **Glorp's copy of `goal-fanout`** lives in `/opt/glorp/bin`. After editing
  `~/harness/bin/goal-fanout`, run `sudo glorp-sync-tools`.
- **Paperclip stays `local_trusted`.** The gate and firewall keep glorp off
  its open API, but any other local user or service on this box still gets
  board access. Switching Paperclip to `authenticated` mode would close that
  for everything, at the cost of a login.
