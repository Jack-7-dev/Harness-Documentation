# Mini PC (orchestrator) — build state

Companion to `HARNESS-PLAN.md`. This box only; the Z440 is tracked separately.

| Thing | Value |
|---|---|
| Host / IP | `charlie-Precision-WorkStation-T3500` / `192.168.1.121` |
| OS | Linux Mint 22.1 (Xia), Ubuntu 24.04 base |
| CPU / RAM | 4 cores, 7 GB |
| Disk | 403 GB free on `/` |
| sudo | **needs a password** — everything below is user-local, no root |
| Role | orchestrator: Paperclip control plane + planner + dispatcher |

## Done (2026-09-04)

**Toolchain** — all user-local, no root:
- Node **24.20.0** via nvm, set as `default` (Paperclip needs ≥24.11; the box had 18)
- pnpm **11.25.0** via corepack
- `jq` 1.7.1 and `uv` 0.12.9 in `~/.local/bin`

**Fixed a live `~/.bashrc` bug.** Line 124 was
`export PATH="$HOME/.local/bin/claude"` — an assignment, not a prepend, so every
interactive shell ended up with *only* that one directory on `PATH`. Now
`…/claude:$PATH`. Backup at `~/.bashrc.bak-20260904`.

**Network link** — `~/.config/systemd/user/z440-tunnel.service`
Runs `ssh -NT -i ~/.ssh/z440_key -L 127.0.0.1:8080:localhost:8080 jack@192.168.1.120`
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
- `~/.ssh/z440_jobd` — **new**, dedicated to the job channel, to be pinned to the
  `jobd` forced command. Host alias `z440-job`. Kept separate from `z440_key` so
  the forced command can't be sidestepped.

**Paperclip** — managed install, `paperclipai` **2026.831.1** in `~/.paperclip`,
server on `127.0.0.1:3100` (loopback), embedded Postgres on `:54329`.
Installed via the CLI directly: the published `install.sh` passes `--no-prompt`,
which this CLI version rejects (upstream mismatch).
Runs as `paperclipai.service` (systemd --user, enabled, linger on) — **not** as a
foreground process, after a stray `timeout` killed the onboarding run once.
Company **Sorbit** `ddeaf594…`, lead agent **Glorp** `c0e87781…` (`claude_local`).
Sorbit is the real product at `~/Sorbit` / `github.com/Sorbit-ltd/Sorbit`.

**Claude quota guard** — `~/harness/bin/claude-quota-guard`
+ `claude-quota-guard.timer` (every 5 min).
Reads `api.anthropic.com/api/oauth/usage` with the local OAuth token and pauses
every `claude_local` agent when a window goes over threshold; resumes only the
agents it paused itself. Default **weekly 50%**, **session 85%**.
Log: `~/harness/logs/quota-guard.log`, state: `~/harness/state/quota-guard.json`.

> See `Z440-SETUP.md` for the executor half of this harness.

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
set status. Includes the plan's 3-strike rule (3 failures → `blocked`).
`--selftest` checks API + SSH; `--dry-run` prints the job JSON without dispatching.

## Corrections to HARNESS-PLAN.md

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

## Still to do on this box

- [x] `paperclipai service install` — done, enabled, linger already on
- [ ] Create the executor agent: adapter `process`, command
      `/home/charlie/harness/bin/z440-exec`, env `Z440_DEFAULT_REPO`,
      `timeoutSec` above the jobd timeout
- [ ] End-to-end run once the Z440 side exists

## Blocked on the Z440 (nothing started there yet)

Verified over SSH: no `agent` user, no `/etc/llama`, `ufw` inactive,
llama-server still binds `0.0.0.0:8080` and serves the raw `.gguf` path as its
model id. Phases 0–3 of `HARNESS-PLAN.md` are all outstanding. The one item that
must come back *here* is the job key:

```
# on the Z440, in /home/agent/.ssh/authorized_keys
command="/home/agent/bin/jobd",no-agent-forwarding,no-X11-forwarding,no-pty ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGBEGEnVcMH45VPntPGu1RzGQh6NE1JIXNYThP9cUzaS minipc-jobd@charlie-T3500
```
