# Mini-PC → Z440 coder harness — build plan

> **This is the original plan (2026-09-04), kept as the record of the reasoning.**
> It has been built, and the build departed from it. For the current state read
> `../README.md`, which points to the current docs;
> where they disagree with this file, they win. The main differences:
> - The executor runs a **64K** window (`--n-cpu-moe 18`), not 32K. So the
>   "32K" constraints and the "Verified facts" table below are historical.
> - **Aider needed no custom adapter.** A Paperclip `process` agent (Zed) runs a
>   bridge script, `z440-exec`. Hermes is not used anywhere.
> - The mini PC → Z440 link is **plain LAN SSH**, not Tailscale.
> - The planner is **Glorp**, Claude Code on DeepSeek V4.1-Flash, behind a
>   spending meter.
> - Test execution, integration branches and an end-of-goal verify agent were
>   added (`FANOUT-PLAN.md`).

Two machines, clean split of duties:

```
  MINI PC  "orchestrator"                 Z440  "executor"  (this box, 192.168.1.199)
  ─────────────────────────               ──────────────────────────────────────────
  planner LLM (big context)               llama-server :8080  Qwen3-Coder-30B-A3B
  scans repos → repo map                  LiteLLM gateway :4000  (auth + alias + log)
  picks next task from goal set   ── ssh ▶ jobd  (forced-command job runner)
  holds the goal/queue state              aider loop, git worktree, runs the tests
  reads back JSON results        ◀──────  returns {status, diff, test output, cost}
```

**The one architectural rule:** the agent loop lives on the Z440, next to the model.
SSH carries a small JSON job in and a small JSON result out — never a token stream.
A 20K-token prompt bounced over the network on every agent step would dominate the
runtime; locally it never leaves the box.

---

## Verified facts about this box (checked 2026-09-04)

| Thing | Value |
|---|---|
| Host / IP | `jack-1-0` / `192.168.1.199`, Ubuntu 24.04.4, 12 cores, 31 GB RAM |
| Model server | `llama.service` active, build 10743, 32K ctx, KV q8_0 |
| Generation speed | **22 tok/s** (measured, 8.6K-token prompt) |
| Prompt processing | **849 tok/s** on ROCm — vs ~75 tok/s on Vulkan, which is why ROCm won |
| GPU / VRAM | **15.92 GiB total**, 14.99 GiB used at 32K, **0.93 GiB free** |
| FA kernels compiled | `GGML_CUDA_FA_ALL_QUANTS=OFF` → only `F16/F16`, `Q4_0/Q4_0`, `Q8_0/Q8_0`, `BF16/BF16` |
| **Tool calling** | **works today** — `--jinja` defaults to ON in this build; verified an OpenAI-shape `tools` request returns `finish_reason: tool_calls` |
| Model id served | the full `.gguf` path (ugly — fix with `--alias`) |
| Auth | none. `--api-key` is supported but unused |
| Exposure | binds `0.0.0.0:8080`, **ufw inactive** → open to the whole LAN |
| Disk | 45 GB free on `/` (repos + worktrees must live within this) |
| Missing tools | `uv`, `tmux`, `pipx`, `tailscale`, `redis` |

Hard constraints the orchestrator must respect: **32K context** (raisable to 64K —
see the measured table below), **one job at a time** (single GPU, 0.93 GiB free),
**first request after a reboot is slow** (one-time HIP JIT compile).

> ⚠️ **KV cache type is not a free parameter on this box.** Only the four combos
> above have compiled flash-attention kernels. Anything else — `q5_1`, `q5_0`,
> `q4_1`, or any asymmetric K/V pair — silently falls off the fast path: **no error,
> no warning**, just prompt processing collapsing from ~800 tok/s to ~8 tok/s and
> degrading as context grows. Measured, twice. Use `q8_0` or `q4_0`, or rebuild with
> `-DGGML_CUDA_FA_ALL_QUANTS=ON`.

---

## Phase 0 — Make the model server agent-grade (30 min)

**Why:** agents hammer the endpoint with retries and need a stable model name; and
right now anything on your LAN can drive the GPU.

1. **Add an alias, an API key, and a slot cap.** Edit the unit:
   ```bash
   sudo systemctl edit --full llama.service
   ```
   Change the `ExecStart` tail to:
   ```
     -t 6 --host 127.0.0.1 --port 8080 \
     --alias qwen3-coder \
     --api-key-file /etc/llama/api-keys \
     --parallel 1 --slots
   ```
   Note `127.0.0.1`: nothing but the local gateway should reach llama-server.
   `--parallel 1` keeps a single 32K KV cache instead of splitting it.

2. **Create the key file:**
   ```bash
   sudo mkdir -p /etc/llama
   openssl rand -hex 32 | sudo tee /etc/llama/api-keys
   sudo chmod 640 /etc/llama/api-keys && sudo chown root:jack /etc/llama/api-keys
   sudo systemctl daemon-reload && sudo systemctl restart llama.service
   ```

3. **Turn on the firewall** (SSH only; do this *from the console or a second SSH
   session* so a mistake doesn't lock you out):
   ```bash
   sudo ufw allow 22/tcp && sudo ufw --force enable && sudo ufw status
   ```

4. **Re-verify tool calling** after the restart (expect a slow first response):
   ```bash
   KEY=$(sudo cat /etc/llama/api-keys)
   curl -s -m 240 localhost:8080/v1/chat/completions -H "Authorization: Bearer $KEY" \
     -H 'Content-Type: application/json' -d '{"model":"qwen3-coder","max_tokens":100,
     "messages":[{"role":"user","content":"list /etc with the tool"}],
     "tools":[{"type":"function","function":{"name":"run_shell","parameters":
     {"type":"object","properties":{"cmd":{"type":"string"}},"required":["cmd"]}}}]}' \
     | jq '.choices[0].message.tool_calls'
   ```
   ✅ Gate: you get a `tool_calls` array back.

---

## Phase 1 — Gateway in front of the model (45 min)

**Why:** LiteLLM gives you per-client API keys, a clean `qwen3-coder` model name,
request/response logging, automatic retries, and a place to later add a second
backend without touching any client.

```bash
# uv is the least painful python installer for this box
curl -LsSf https://astral.sh/uv/install.sh | sh
exec $SHELL -l
uv tool install "litellm[proxy]"
```

`/etc/litellm/config.yaml`:
```yaml
model_list:
  - model_name: qwen3-coder
    litellm_params:
      model: openai/qwen3-coder
      api_base: http://127.0.0.1:8080/v1
      api_key: os.environ/LLAMA_KEY
      timeout: 900
      max_retries: 2
litellm_settings:
  drop_params: true
general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

Run it as a systemd unit on `127.0.0.1:4000` (same pattern as `llama.service`:
`User=jack`, `Restart=on-failure`, `EnvironmentFile=/etc/litellm/env`).

✅ Gate: `curl 127.0.0.1:4000/v1/models -H "Authorization: Bearer $LITELLM_MASTER_KEY"`
lists exactly `qwen3-coder`.

> Skip this phase if you want the shortest possible path — Aider can talk to
> llama-server directly. You lose logging and key rotation.

---

## Phase 2 — The executor: user, workspace, agent (1–2 h)

**Why:** the model is going to run code it wrote. Give it its own account and its
own copy of the repo so a bad step can't touch your home directory or your branches.

1. **Dedicated user + workspace**
   ```bash
   sudo adduser --disabled-password --gecos "" agent
   sudo -u agent mkdir -p /home/agent/{work,jobs,logs}
   sudo apt install -y tmux
   ```
   Give `agent` **no sudo**. If a repo's test suite needs system packages, install
   them yourself, once.

2. **Install Aider as the executor** (git-native, non-interactive mode, builds its
   own repo map, commits its own work):
   ```bash
   sudo -u agent bash -lc 'curl -LsSf https://astral.sh/uv/install.sh | sh'
   sudo -u agent bash -lc '~/.local/bin/uv tool install aider-chat'
   ```

3. **Point it at the local model** — `/home/agent/.aider.conf.yml`:
   ```yaml
   model: openai/qwen3-coder
   auto-commits: true
   yes-always: true
   stream: false
   map-tokens: 2048        # keep the repo map small — 32K total context
   cache-prompts: true
   ```
   and `/home/agent/.env`:
   ```
   OPENAI_API_BASE=http://127.0.0.1:4000/v1
   OPENAI_API_KEY=<litellm key>
   ```

4. **Smoke test by hand** before automating anything:
   ```bash
   sudo -u agent -i
   git clone <some throwaway repo> ~/work/scratch && cd ~/work/scratch
   aider --message "add a docstring to the main entry point"
   ```
   ✅ Gate: Aider edits a file and makes a git commit, unattended.

**Alternative worth knowing:** [OpenHands](https://github.com/All-Hands-AI/OpenHands)
is the heavier option — full sandboxed agent in Docker, browses, runs long
multi-file tasks. It leans much harder on function-calling quality than Aider does,
and a 30B-A3B at Q4 with 32K context will struggle with its long system prompts.
Start with Aider; graduate to OpenHands only if you hit Aider's ceiling.

---

## Phase 3 — The job channel (SSH, locked down) (1 h)

**Why:** the mini PC's key should be able to run *one* program — the job runner —
not an arbitrary shell. This is what makes the whole design safe to leave running.

1. **`/home/agent/bin/jobd`** — reads one JSON job on stdin, writes one JSON result
   on stdout. Sketch:
   ```bash
   #!/usr/bin/env bash
   set -euo pipefail
   JOB=$(cat)                                  # {"repo","branch","task","tests"}
   ID=$(date +%s)-$RANDOM
   exec 9>/home/agent/.lock; flock -w 7200 9   # single-flight: one GPU
   repo=$(jq -r .repo <<<"$JOB"); task=$(jq -r .task <<<"$JOB")
   wt=/home/agent/work/$ID
   git -C /home/agent/repos/$repo worktree add -b agent/$ID "$wt"
   cd "$wt"
   timeout 3600 aider --message "$task" >/home/agent/logs/$ID.log 2>&1 || true
   tests=$(jq -r '.tests // empty' <<<"$JOB")
   [ -n "$tests" ] && timeout 900 bash -c "$tests" >>/home/agent/logs/$ID.log 2>&1
   rc=$?
   jq -n --arg id "$ID" --arg br "agent/$ID" --argjson rc "${rc:-0}" \
         --arg diff "$(git diff main...HEAD --stat)" \
         --arg log "$(tail -c 8000 /home/agent/logs/$ID.log)" \
         '{id:$id,branch:$br,exit:$rc,diff:$diff,log:$log}'
   ```
   Key properties: **one job at a time** (`flock`), **its own worktree and branch**,
   **hard timeouts**, **truncated log** so the result stays small.

2. **Forced command** in `/home/agent/.ssh/authorized_keys`:
   ```
   command="/home/agent/bin/jobd",no-agent-forwarding,no-X11-forwarding,no-pty ssh-ed25519 AAAA...minipc
   ```
   Now `ssh agent@z440 < job.json > result.json` is the *entire* API surface, and
   that key can do nothing else.

✅ Gate: from the mini PC, `echo '{"repo":"scratch","task":"add a README"}' | ssh agent@192.168.1.199 | jq .`
returns a result object and a new `agent/<id>` branch exists on the Z440.

---

## Phase 4 — Network link (30 min)

**Why:** LAN IPs move, and you'll eventually want this from outside the house.

**Recommended — Tailscale** on both boxes: stable name `z440`, encrypted, no ports
forwarded, ACL-able down to "mini PC may reach z440:22 only".
```bash
curl -fsSL https://tailscale.com/install.sh | sh && sudo tailscale up --ssh=false
```

**If you'd rather not add a dependency:** keep the LAN IP and, when you need the
raw model endpoint from the mini PC (e.g. to run the *planner* on Qwen too), use a
tunnel rather than opening the port —
`autossh -M 0 -N -L 4000:localhost:4000 jack@z440` under systemd.

---

## Phase 5 — The orchestrator on the mini PC (the real work)

Four small pieces. Keep each one boring.

1. **Repo scanner** — clone/pull the repos it manages, then produce a *compact*
   digest per repo: `git log --oneline -20`, changed-file churn, `tree -L 3`, plus a
   symbol map (`aider --show-repo-map`, or `ctags -R --output-format=json`). Cache
   it; regenerate on new commits only.

2. **Goal store** — one SQLite file (or `goals.yaml` + `tasks.jsonl`, versioned in
   git). Rows: `goal → tasks → status → result branch`. This is the memory of the
   system; it must survive both machines rebooting.

3. **Planner LLM** — given (goal, repo digest, last N results) emit **one** next
   task as JSON matching the `jobd` contract. Model choice:
   - *Cloud API* (Claude/GPT) — best planning quality, and the mini PC needs no GPU.
     Recommended default: planning is low-volume, so it's cheap.
   - *Local* — a **Hermes 4** build (Nous Research) is a reasonable local
     planner/tool-caller if the mini PC has the VRAM; a small Qwen3 instruct also
     works. Below ~14B, plan quality degrades fast — measure before committing.
   - *Reuse the Z440 model* over the tunnel — free, but it stalls execution while it
     plans, since there's only one GPU. Fine for a first version.

   Prompt the planner with the constraints explicitly: **32K context on the executor,
   one task at a time, tasks must be single-PR sized, always specify a test command.**

4. **Dispatcher loop** — pop task → `ssh agent@z440 < job.json` → store result →
   feed result back to the planner → repeat. Add: 3-strike rule (a task failing
   three times goes to a human-review queue), and a nightly `git fetch` of agent
   branches so you can review diffs on the mini PC.

> **Both Hermes Agent and Paperclip are real and relevant here — see the evaluation
> section below.** Short version: **Paperclip replaces almost all of this phase**, so
> read that before writing any of it. Hermes can run on the Z440 too, but only after
> you buy 64K of context at a measured cost.

---

## Evaluation: Hermes Agent and Paperclip

Researched and benchmarked on this box, 2026-09-04. Both are real, both are
MIT/open-source, and both map onto this design — Paperclip as the control plane on
the mini PC, Hermes as an optional agent runtime on the Z440. They are complementary
layers, not competing choices.

### Paperclip — ✅ use it, as the mini-PC orchestrator

[github.com/paperclipai/paperclip](https://github.com/paperclipai/paperclip) — a
Node.js + React control plane that models agents as employees in an org chart:
roles, reporting lines, tickets, **per-agent monthly budgets with auto-pause at
100%**, heartbeat scheduling, and human approval gates. MIT, self-hosted,
`docker compose up`.

**This is Phase 5, already built.** The goal store, the dispatcher loop, the
3-strike escalation, the review queue — Paperclip has all of it, plus budget
controls and an audit trail I wasn't going to hand-roll.

**Requirements** (all fine on a mini PC): Node 24.11+, pnpm 9.15+, embedded
Postgres, single Node process.

**Adapters — the important part.** Paperclip's *process* adapter ships only
`claude_local` and `codex_local`, `spawn()`s children **on the Paperclip host**, and
has no generic "run any command" mode ("not yet implemented in V1"). So it cannot
reach the Z440. But you do **not** need a custom adapter, because since
**v2026.626.0 Paperclip ships Hermes as a first-class built-in**:

| adapter | what it does |
|---|---|
| `hermes_local` | runs Hermes Agent on the Paperclip host (`hermes chat -q …`) |
| **`hermes_gateway`** | **calls a *remote* Hermes HTTP API — this is the mini-PC → Z440 hop** |

`hermes_gateway` config: `apiBaseUrl`, `apiKey` (Hermes's `API_SERVER_KEY`, stored as
a Paperclip secret), `sessionKeyStrategy` (default `issue`), `timeoutSec` (default
600). The remote box must run Hermes with its HTTP API enabled, serving `/v1/runs`,
`/v1/runs/{id}`, `/v1/runs/{id}/events`. Paperclip's own docs say loopback HTTP is
always allowed and **non-loopback should go over HTTPS or a private network like
Tailscale** — which is exactly Phase 4.

**Cost of adopting Paperclip: zero custom code**, if you run Hermes on the Z440.
Only if you keep **Aider** as the executor do you need a custom adapter
(`ServerAdapterModule` from `@paperclipai/adapter-utils`: `type`, `execute()`,
`testEnvironment()`, registered in `server/src/adapters/registry.ts`; the docs ship
an HTTP/webhook example) wrapping the `jobd` call from Phase 3 — roughly 100 lines.

**Known open bug** ([#82](https://github.com/NousResearch/hermes-paperclip-adapter/issues/82)):
the stock Hermes agent improvises `curl | python3` calls, Hermes's own security
scanner flags them HIGH risk, and once denied the refusal *persists across
heartbeats* — the agent never recovers. Workaround is editing the prompt template to
mandate the Paperclip MCP tools instead of shell pipes. Budget half a day for this.

### Hermes Agent — 🟡 runs on the Z440, but you must buy the context

[NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) — a
model-agnostic agent harness with profiles (each with its own `config.yaml`,
`SOUL.md` identity, SQLite memory, cron jobs, skills, MCP config), persistent
"holographic" memory with FTS5 search, and support for any OpenAI-compatible
`base_url`.

**The obstacle: Hermes enforces a hard minimum of 64,000 tokens for
`model.context_length` and rejects the model at startup.** This box serves 32K.
The tracking issue asking for a warn-instead-of-fail flag
([#53347](https://github.com/NousResearch/hermes-agent/issues/53347)) is **open with
no maintainer response** — don't plan around it landing.

**Do not use the popular workaround.** Declaring `context_length: 64000` while the
server serves 32K is actively harmful: Hermes fires compression at a *percentage* of
the context it believes it has, so it will build a 40K prompt and llama-server will
silently truncate it — an agent that quietly forgets the middle of its own task.

#### What 64K actually costs — measured 2026-09-04

Non-KV footprint is **13.40 GiB**; each `--n-cpu-moe` layer moves ~324 MiB of expert
weights (128 experts × 3 × 2048 × 768 = 604M params/layer) off the GPU.

| config @ 64K | KV size | free VRAM | prompt proc | gen | result |
|---|---|---|---|---|---|
| q8_0, `-ncmoe 12` | 3.188 G | — | — | — | **OOM** — died on a 3264 MiB alloc |
| q5_1, `-ncmoe 12` | 2.250 G | 0.32 G | ~8 tok/s | — | **unusable** |
| q5_1, `-ncmoe 17` | 2.250 G | 1.90 G | ~8 tok/s | — | **unusable even with headroom** |
| **q8_0, `-ncmoe 20`** | 3.188 G | 1.84 G | **650 tok/s** | **20** | ✅ **recommended** |
| q4_0, `-ncmoe 14` | 1.688 G | 1.44 G | 782 tok/s | 20 | ✅ works, lossy KV |
| *32K q8_0 `-ncmoe 12`* | *1.594 G* | *0.93 G* | *849* | *22* | *today* |

Three conclusions, each of which contradicts an intuition worth naming:

1. **KV quantization cannot buy you 64K on this build.** `q5_1` failed identically
   at 0.32 GiB and 1.90 GiB free — it isn't a memory problem, it's that `q5_1` has
   **no compiled FA kernel** here (`GGML_CUDA_FA_ALL_QUANTS=OFF`). It fails silently.
2. **Pushing MoE layers to CPU is the cheap lever, not the expensive one.** Eight
   extra CPU layers cost **−23% prompt processing and −9% generation** — and keep KV
   lossless. That is the recommended path.
3. **Lowering the weight quant is dominated — don't.** Both working configs reach
   64K without touching weights, so a Q3 requantize buys nothing while degrading the
   one thing doing the work (code gen retains ~98% of FP16 at Q8, ~95% at Q5, ~92% at
   Q4; MoE is *more* quant-sensitive since each expert is only ~4.7M params), throws
   away the UD dynamic quant, and costs a ~13 GB re-download.

**Best available config costs one recompile:** build with
`-DGGML_CUDA_FA_ALL_QUANTS=ON` to unlock asymmetric **`K=q8_0 / V=q4_0`**. K drives
attention scores and long-range recall; V tolerates more loss. That lands between the
two working configs on both speed and quality.

#### ⛔ But 64K is the worst value to run Hermes at — don't buy it

Hermes computes its compression trigger as
`threshold_tokens = max(int(model_context * threshold), MINIMUM_CONTEXT_LENGTH)`
with `MINIMUM_CONTEXT_LENGTH = 64_000` hardcoded
([#31600](https://github.com/NousResearch/hermes-agent/issues/31600)). The floor
beats the percentage at small context sizes:

| model context | `0.5 × context` | actual trigger | window left after it fires |
|---|---|---|---|
| **65,536** | 32,768 | **64,000** (floor wins) | **1,536 tokens** |
| 131,072 | 65,536 | 65,536 (percentage wins) | 65,536 |

At 64K compression fires at **97.6%** of the window — and tool-call JSON starts
truncating at **~60K**, *before* compression can ever run. The reported failure mode
is a silent loop: malformed JSON parses to an empty object, a warning is logged, no
exception raised, and the agent retries the same memory operation every turn with a
growing context, indefinitely.

**64K is simultaneously the minimum that passes Hermes's startup check and the
maximum that sits inside its deadlock zone.** The framework assumes ~128K, where the
percentage wins and compression works as designed. On this card 128K means a 6.375
GiB KV cache and ~29 of 48 MoE layers on CPU — extrapolating the measured curve
(~−25 tok/s pp per offloaded layer) puts prompt processing near **425 tok/s**, half
of today's. Not measured; verify before relying on it.

**Where the context actually goes:** ~16,300 tokens of fixed overhead per request,
~16K of it eagerly-injected JSON schemas for 60+ tools, whether used or not.
Disabling unused toolsets reportedly cuts this to ~490 tokens — but that does not
help, because the startup guard tests `model_context`, not overhead.
*(Overhead figures are third-party analysis, not Nous docs — approximate.)*

### Verdict

| Component | Role | Verdict |
|---|---|---|
| **Paperclip** | mini PC — orchestrator, budgets, approvals | ✅ **adopt** — independent of everything else, costs no VRAM |
| **Aider** | Z440 — executor at 32K | ✅ **start here** — fastest config (849/22), built for this budget |
| **Hermes Agent** | Z440 — executor at 64K | ⛔ **don't** — 64K is inside its compression deadlock zone |
| **Hermes Agent** | mini PC — planner on a cloud model | ✅ **best home for it** — 128K+ free, all features, no GPU cost |
| **`hermes_gateway`** | the mini PC → Z440 hop | ✅ **built in** — no custom adapter needed |
| **Lower weight quant** | — | ⛔ **rejected** — dominated; degrades code gen for nothing |
| **`q5_1`/`q5_0`/`q4_1` KV** | — | ⛔ **broken on this build** — silent 100× slowdown |
| **LiteLLM** | Z440 — gateway | ✅ still worth it (keys, logging, alias) |

**Paperclip and Hermes are not alternatives** — they are different layers, a control
plane and an agent runtime. "Both" is coherent; "one or the other" mostly isn't.

**Recommended sequence:**

1. **Paperclip on the mini PC now.** Independent, zero VRAM cost, replaces the whole
   hand-rolled Phase 5.
2. **Keep 32K + Aider as the executor** while you get the loop working end to end.
   It is the fastest configuration you have and Aider is designed to work in that
   budget via explicit file selection and a small repo map. Costs one custom adapter.
3. **If you want Hermes's memory / profiles / cron, run it on the mini PC** against
   a cloud model, where 128K+ context is free. You get every Hermes feature, the
   Z440 never leaves its fastest configuration, and you stay far outside the 64K
   deadlock zone. Budget time for bug #82 either way.

**Do not put Hermes on the Z440 at 64K.** The 64K figure is a hardcoded startup guard
in someone else's framework, not a need of your workload — and satisfying it lands
you exactly in the regime where its own compression logic is broken. The Z440's job
is executing code fast at 32K; Hermes's job, if you want it, is thinking on the box
that has context to spare.

*(If you ever do want Hermes GPU-side, target ~128K rather than 64K — and measure
`-ncmoe 29` first, since the extrapolation above is unverified.)*


---

## Phase 6 — Ops & acceptance

**Make it survive reboots**
- `llama.service` ✅ already enabled · add `litellm.service` and (if used)
  `autossh-tunnel.service`, both `WantedBy=multi-user.target`.
- Warm the GPU on boot so the first *real* job doesn't eat the JIT cost: a
  `llama-warmup.service` with `After=llama.service` that POSTs a 5-token request.

**Watch these**
```bash
journalctl -u llama.service -f        # model server
tail -f /home/agent/logs/<id>.log     # a job's agent transcript
df -h /                               # 45 GB free — worktrees add up
```
Add a cron that prunes worktrees older than 7 days and merged `agent/*` branches.

**Acceptance ladder — don't skip a rung**
1. `curl` returns a `tool_calls` response with the API key set. *(Phase 0)*
2. LiteLLM lists `qwen3-coder` and proxies a completion. *(Phase 1)*
3. Aider makes an unattended commit in a scratch repo. *(Phase 2)*
4. One JSON job over SSH produces a branch + result JSON. *(Phase 3)*
5. Mini PC runs that same job over Tailscale. *(Phase 4)*
6. Planner turns a one-line goal into a valid job JSON. *(Phase 5)*
7. **End-to-end:** set a goal like *"add type hints to module X"*, walk away, come
   back to 3+ reviewable `agent/*` branches with passing tests.

---

## Suggested order & effort

| # | Phase | Effort | Blocks |
|---|---|---|---|
| 0 | Harden + alias the model server | 30 m | everything |
| 2 | Agent user + Aider (do before 1 — proves the loop) | 1–2 h | 3 |
| 3 | `jobd` + forced-command SSH | 1 h | 5 |
| 4 | Tailscale | 30 m | — |
| 1 | LiteLLM gateway | 45 m | — (optional) |
| 5 | Orchestrator (Paperclip; + adapter only if executor is Aider) | 4–8 h | — |
| 6 | systemd, warmup, pruning | 1 h | — |

Fastest useful milestone: **Phases 0 → 2 → 3**, about half a day, gets you
"mini PC sends a task string, Z440 returns a reviewable branch."
