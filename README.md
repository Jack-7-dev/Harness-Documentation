# Sorbit coder harness

A goal written in Paperclip is split into small units by a cloud planner, and
each unit is edited by a local Qwen3-Coder model on a GPU box. Tests and checks
run after every unit, and the finished branch comes back to a human for review.

```
 Paperclip (mini PC) ──▶ GLORP    plans (Claude Code on DeepSeek V4.1-Flash, metered)
                            │     goal-fanout: integration branch + one issue per unit
                            ▼
                         ZED      one unit at a time ── ssh z440-job ──▶ jobd → aider → Qwen3-Coder
                            ▼
                         VERITY   end-of-goal suite ── ssh z440-verify ──▶ verifyd (no GPU)
                            ▼
 human review ◀── goal "in_review"
```

## The two machines

| Box | Address | Runs |
|---|---|---|
| **Mini PC** `charlie-Precision-WorkStation-T3500` | 192.168.1.121, tailnet 100.114.50.113 | Paperclip :3100, the agent bridges in `~/harness/bin`, DeepSeek meter, Glorp's isolated user |
| **Z440** `jack-1-0` | 192.168.1.199 (static), LAN only | `llama.service` (Qwen3-Coder-30B-A3B, 64K) :8080, LiteLLM :4000, the sandboxed `agent` account with `jobd` / `verifyd` |

## Which doc

| Doc | Read it for |
|---|---|
| [`ENGINEER-GUIDE.md`](ENGINEER-GUIDE.md) | Giving Glorp work and reviewing the result. For engineers, not admins. |
| [`Z440.md`](Z440.md) | The executor as built: model server and how to operate it, tuning and sampling, aider, `jobd`, the Paperclip bridge, context and speed measurements |
| [`MINIPC.md`](MINIPC.md) | The orchestrator as built: Paperclip and its agents, quota guard, and **Glorp on DeepSeek** (meter, caps, isolation) |
| [`FANOUT.md`](FANOUT.md) | Goal → units → verify → loop-back as built, and changes since |
| [`history/`](history/) | The original plans (`HARNESS-PLAN.md`, `FANOUT-PLAN.md`), kept for their reasoning and measurements. **Not current.** Where they disagree with the docs above, the docs above win. |

## Health at a glance

```bash
# mini PC
systemctl --user status paperclipai z440-tunnel
curl -s localhost:4100/_meter/status | jq '{open, windows}'
curl -s -o /dev/null -w '%{http_code}\n' localhost:3100/api/health   # Paperclip
journalctl -u z440-sorbit-push -n 3 -o cat                          # Z440 base sync

# Z440
ssh z440 'systemctl is-active llama litellm; curl -s localhost:8080/health; df -h / | tail -1'
```
