# Working in this repo

This repo is **documentation only** — it is the written record of the Sorbit
coder harness that runs on the mini PC and the Z440. No harness code lives here.

**Read [`ONBOARDING.md`](ONBOARDING.md) before doing anything with the harness.**
It is self-contained: architecture, the lifecycle of a goal, the job contracts,
the trust boundaries, the invariants, the diagnostic commands and the known open
items. Do not rediscover it by probing the boxes.

## The live systems are not yours to change

Everything described here runs in production on two machines. Unless the user
explicitly asks you to change it:

- **Look, don't change:** `~/harness`, `~/.paperclip`, `/etc/deepseek-meter`,
  systemd units, and anything under `/home/agent` on the Z440.
- **Never** `npm ci`, `git checkout`, or commit inside the Z440's repos — the
  agents share those checkouts.
- **Never** paste secrets or customer data into a Paperclip issue. Glorp's text
  goes to DeepSeek's API.
- Don't change the `charlie` password or add SSH keys.

When you are asked to change the harness, follow
[§15 of ONBOARDING.md](ONBOARDING.md#15-changing-the-harness-safely): idle board,
`.bak-<prior-state>` backup, test on `scratch` or a throwaway, drop-in rather
than edit for managed units, and append a dated entry to `FANOUT.md`'s
**"Since this was written"** saying what broke, how it was proved, and which
backup file marks it.

## Documentation conventions

- Docs describe **what is actually deployed**, with the measurements and the
  reasoning that chose it. Where a plan in `history/` disagrees, the current docs
  win.
- `FANOUT.md`'s "Since this was written" is the dated change log and the last
  word on current behaviour. Earlier prose in the same doc may describe what a
  later entry replaced.
- State findings honestly, including what was *not* verified. Several entries
  exist specifically to record that a check has never fired or a path has never
  run live.
- Commit messages are one descriptive line in the imperative
  (`git log --oneline` for the house style).
