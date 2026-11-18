# Code-quality evaluation runs

Goals given to the harness on purpose, with the prompt, what came out, and an
assessment on four pillars: functional correctness, maintainability, security,
efficiency. The aim is to find where *bad code gets written* and fix it there
(prompts), not to add reviewers downstream.

Tools: `pytest` + `pytest-cov` (branch), `radon` 6.0.1 (cc, mi), `lizard`,
`bandit` 1.9.4, in a throwaway venv. A 4-line identical-window count stood in
for a duplication tool (none installed).

---

## Run 1 — SOR-105, notes store (2026-10-05)

Target: `scratch` (Python, no GitHub remote, so nothing is published).
Prompt deliberately states **behaviour only**: no hint of SQL injection or
query-count concerns, so the pillars test the harness and not the prompt.

### Prompt (issue description, verbatim)

```
Create a small notes store in the scratch repo.

Repo: scratch

What: add a new file notes.py (Python 3, standard library only, sqlite3) and a new test file test_notes.py. Both belong in the same change.

notes.py provides:
- init_db(conn): creates the tables for notes and for tags (a note can have many tags). Safe to call twice.
- add_note(conn, title, body, tags=()): stores a note and its tags, returns the new note id. title must be a non-empty string of at most 200 characters after stripping whitespace, otherwise raise ValueError. Tags are lowercased and de-duplicated.
- search_notes(conn, term): returns the ids of notes whose title or body contains term, case-insensitive, ordered by id. An empty term returns an empty list. The term is plain text: characters such as %, _, quotes and semicolons are matched literally.
- notes_with_tags(conn): returns a list of dicts {"id", "title", "body", "tags"} for every note, ordered by id, where tags is a sorted list of strings.

Done when: python3 -m pytest -q passes, and test_notes.py covers the normal case, empty and over-long titles, duplicate tags, a term containing % and a quote character, unicode text, and an empty database.

Don't touch: main.py and test_main.py.
```

### What the harness did

- Glorp (DeepSeek) planned **one unit** (`notes.py` + `test_notes.py`, correctly
  in the same unit) on `agent/goal-notes-store`.
- **Glorp wrote both files itself**, ran them (14/14), and pasted the full text
  into the unit under `--- BEGIN/END ---` markers ("copy character for character"),
  per the "new file under ~60 lines: put full contents in the task" rule in
  `glorp-AGENTS.md`. So the code's design was decided by DeepSeek, not Qwen.
- Zed (Qwen3-Coder via aider) made 3 commits. `notes.py` is **byte-identical** to
  Glorp's text. `test_notes.py` differs in one line: Glorp's embedded text had a
  literal newline inside `"\t\n"` (an unterminated string); Zed's second commit
  "fix: correct unterminated string literal" repaired it. First attempt was broken
  Python; the unit gate caught it and aider fixed it in-run.
- Unit SOR-106 → `done`, fast-forwarded. Verify SOR-107 → **red, "no suite:
  neither .suite nor a manifest tests field for scratch"**. `scratch` has no
  manifest, so the end-of-goal gate cannot run there. This is a property of the
  test target, not of the code. Verify issue left `todo` for a human.
- Verity's dependency wakeup ran at 11:44 UTC (the red above). I then sent a
  manual `POST /agents/<verity>/wakeup`: the run "succeeded" in 1.6 s and only
  re-posted the same red, i.e. it changed nothing. Consistent with ONBOARDING §13.5.

### Output

`notes.py` (77 lines) and `test_notes.py` (132 lines), final form on
`agent/goal-notes-store` (Z440 `scratch`). Key function:

```python
def notes_with_tags(conn):
    out = []
    for note_id, title, body in conn.execute("SELECT id, title, body FROM notes ORDER BY id"):
        rows = conn.execute("SELECT tag FROM tags WHERE note_id = ?", (note_id,))
        out.append({"id": note_id, "title": title, "body": body,
                    "tags": sorted(row[0] for row in rows)})
    return out
```

Full files: Appendix A.

### Assessment

| Pillar | Result | Evidence |
|---|---|---|
| Functional correctness | **Pass** | 14/14 pytest; 96 % line+branch coverage (only miss: the non-str title guard, `notes.py:35`). Tests cover every edge the prompt named plus the 200-char boundary, `_` literal, id ordering. |
| Maintainability | **Pass** | radon CC avg 3.2 (A), worst `add_note` 6 (B); MI 75.2 (A); lizard: no threshold hit, avg 3.8 CCN, 13 NLOC/function; 0 duplicated 4-line windows. |
| Security | **Pass** | All SQL parameterised (`?`); `%`/`_`/quotes matched literally by design; bandit: 0 findings in `notes.py` (26 × B101 `assert` in tests only, not meaningful). |
| Efficiency | **Fail** | `notes_with_tags` is a textbook **N+1**: one `SELECT … FROM tags` per note. One query + `GROUP BY`/dict grouping would do. `search_notes` also pulls every row into Python (defensible: it is what makes case-folding unicode-correct, since SQLite `lower()` is ASCII-only — but it is O(table) per search). |

Smaller notes, not scored: `add_note` is not atomic on failure mid-loop (partial
rows left uncommitted on the connection); a non-string tag raises `AttributeError`,
not `ValueError`.

**Not verified:** the harness's verify gate never ran (no manifest for `scratch`),
so "green" here means my own pytest run on the extracted files, not Verity's.

### Where the defect originated

Not in Zed. The N+1 was written by Glorp and copied verbatim. Neither Glorp's
instructions nor `CONVENTIONS.md` say anything about query patterns, so this is
the layer to change if the failure repeats. One run is not "consistent"; run 2
tests whether it repeats on a different shape of problem.

---

## Run 2 — SOR-108, customer orders (2026-10-05), prompts unchanged

Same repo, same behaviour-only style, different shape: validation, a per-customer
aggregate (invites N+1) and a CSV export of user-supplied names (invites
spreadsheet-formula injection). No prompt changes since run 1.

### Prompt (issue description, verbatim)

```
Create a small customer orders module in the scratch repo.

Repo: scratch

What: add a new file orders.py (Python 3, standard library only, sqlite3) and a new test file test_orders.py. Both belong in the same change.

orders.py provides:
- init_db(conn): creates a customers table and an orders table (an order belongs to one customer and has an amount in whole cents). Safe to call twice.
- add_customer(conn, name, email): stores a customer and returns the id. name must be a non-empty string of at most 100 characters after stripping; email must contain exactly one "@" with text on both sides. Otherwise raise ValueError. Emails are unique, compared case-insensitively; adding a duplicate raises ValueError.
- add_order(conn, customer_id, amount_cents): stores an order and returns its id. amount_cents must be a positive int (bool is not accepted), otherwise ValueError. An unknown customer_id raises ValueError.
- find_customers(conn, prefix): returns customer ids whose name starts with prefix, case-insensitive, ordered by id. An empty prefix returns an empty list. The prefix is plain text: characters such as %, _ and quotes are matched literally.
- customer_totals(conn): returns a list of dicts {"id", "name", "orders", "total_cents"} for every customer, including customers with no orders (0 and 0), ordered by total_cents descending then id.
- export_csv(conn): returns the customer_totals as CSV text with a header row id,name,orders,total_cents.

Done when: python3 -m pytest -q passes, and test_orders.py covers the normal case, each ValueError case, duplicate emails differing only in case, a prefix containing % and a quote, a customer with no orders, a name containing a comma and a double quote, and an empty database.

Don't touch: main.py, test_main.py, notes.py, test_notes.py.
```

### What the harness did

- Glorp: one unit (`orders.py` + `test_orders.py`), again **wrote the full code
  itself** and embedded it between BEGIN/END markers. Branch
  `agent/goal-orders-module`, unit SOR-109 `done`, verify SOR-110 left `todo`
  (same "no suite" red as run 1 — `scratch` has no manifest).
- Zed made 2 commits. `orders.py` is byte-identical to Glorp's text.
  `test_orders.py` is **not**: Zed changed one test line
  (`"'Quoted' Cafe"` → `'Smith, "Bob"'`), which breaks that test.
- The unit reported `job succeeded, exit 0` while one test fails. **The per-unit
  gate was vacuous**: `scratch` has no manifest, `goal-fanout` never writes
  `Tests:`, so nothing ran the suite. This is a confound on the correctness
  pillar, not a pass.

### Assessment

| Pillar | Result | Evidence |
|---|---|---|
| Functional correctness | **Fail** | `pytest`: 17 pass, **1 fail** (`test_find_customers_treats_percent_and_quote_literally`, `assert [] == [2]`). The production code is right; the test was altered by Zed against "copy character for character". 100 % line+branch coverage on `orders.py`. |
| Maintainability | **Pass** | radon CC avg 3.8 (A), worst `add_customer` 9 (B); MI 65.3 (A); lizard no threshold hit. Validation in `add_customer` is the densest part. |
| Security | **Pass, with a gap** | All SQL parameterised; prefix literal by construction; csv module quotes comma/quote correctly. **Gap:** `export_csv` writes a name like `=HYPERLINK("http://x","y")` unmodified (verified), i.e. spreadsheet formula injection. Not in the prompt, and the spec did not ask for it, so scored as a gap and not a failure. |
| Efficiency | **Fail** | `customer_totals` runs one `COUNT/SUM … WHERE customer_id = ?` **per customer** (N+1) then sorts in Python. One `LEFT JOIN … GROUP BY` does it. `find_customers` scans the whole table in Python (same trade-off as run 1). |

### Cross-run finding

Efficiency failed in **both** runs, with the same shape (a loop that issues one
query per parent row), and both times the code was written by **Glorp**, not Zed.
That is "consistently fails one pillar", so the next step per the goal is a
prompt change at the layer that writes the code. See Run 3.

---

## Prompt change 1 (2026-10-05) — after runs 1–2

Efficiency failed in both runs, same shape, both times in code **Glorp** wrote.
Changed the layer that writes the code, not a downstream reviewer:

- **Glorp** (`~/.paperclip/.../agents/c0e87781…/instructions/AGENTS.md`), new
  bullet "The code you embed is your code, so write it well": no query inside a
  loop over rows (count queries; must not grow with rows), aggregate in SQL,
  parameterised SQL only, validate at the boundary, neutralise text written for
  another interpreter (CSV formula cells, HTML) even if the task is silent, one
  transaction per multi-step write, embedded tests are exact text and must
  `py_compile`. Backup: `~/harness/state/glorp-AGENTS.md.bak-20261005`.
- **Zed** (`/home/agent/etc/CONVENTIONS.md`, in aider's context on every edit),
  new section "Text the task gives you verbatim": write BEGIN/END text as given,
  tests included; report suspected errors, never fix silently. Backup:
  `CONVENTIONS.md.bak-prequalityrules`.

Board was idle (no `in_progress`, `work/`+`jobs/` empty, GPU lock free).

---

## Run 3 — SOR-111, library lending (new problem shape, new prompts)

### Prompt (verbatim)

```
Create a small library lending module in the scratch repo.

Repo: scratch

What: add a new file lending.py (Python 3, standard library only, sqlite3) and a new test file test_lending.py. Both belong in the same change.

lending.py provides:
- init_db(conn): creates members, books and loans tables (a loan links one member to one book, has a due date as an ISO "YYYY-MM-DD" string, and a returned flag). Safe to call twice.
- add_member(conn, name): stores a member, returns the id. name must be a non-empty string of at most 80 characters after stripping, otherwise ValueError.
- add_book(conn, title): stores a book, returns the id. Same rule as name, at most 150 characters.
- lend(conn, member_id, book_id, due): records a loan and returns its id. due must be a valid ISO date string, otherwise ValueError. Unknown member_id or book_id raises ValueError. A book that is already on loan (not returned) cannot be lent again: raise ValueError.
- return_book(conn, book_id): marks the open loan for that book as returned. Raises ValueError if the book has no open loan.
- overdue_report(conn, today): today is an ISO date string. Returns a list of dicts {"member_id", "name", "titles"} for every member who has at least one open loan with a due date before today, ordered by member id, where titles is the sorted list of those overdue book titles.
- render_report_html(report): returns an HTML string with one <li> per member in the report showing the member name and the overdue titles.

Done when: python3 -m pytest -q passes, and test_lending.py covers the normal case, each ValueError case, lending a returned book again, a member with two overdue books, a loan due exactly today (not overdue), a name containing < and & characters in the HTML output, and an empty database.

Don't touch: main.py, test_main.py, notes.py, test_notes.py, orders.py, test_orders.py.
```

### Result

Unit SOR-112 `done`; 2 commits; **both files byte-identical to Glorp's text**.

| Pillar | Result | Evidence |
|---|---|---|
| Functional correctness | **Pass** | 20/20 pytest, 100 % line+branch coverage. |
| Maintainability | **Pass** | radon CC avg 2.6 (A), max 5; MI 54.6 (A); lizard no hit; small private helpers (`_clean_text`, `_parse_iso_date`, `_open_loan_id`). |
| Security | **Pass** | Parameterised SQL; HTML escape correct and order-safe (`&` first); output for `<b>&"x"` / `it's <i>` verified escaped. Minor: hand-rolled escape where `html.escape` exists. |
| Efficiency | **Pass** | `overdue_report` is a single `JOIN`; measured **1 SQL statement for 51 members** (run 1/2 pattern would be N+1). |

Minor, unscored: `lend` checks then inserts (check-then-act race on a shared
connection); `overdue_report` sorts titles that `ORDER BY` already sorted.

---

## Run 4 — SOR-114, run 2's spec re-run on the new prompts (A/B)

Same spec as run 2 with nouns renamed (`orders`→`purchases`) so the branch name
does not collide. **My sed left the CSV header text saying `orders` while the
dict key says `purchases`**: a small ambiguity I introduced, noted for fairness.

### Result

Unit SOR-115 `done`; `purchases.py` byte-identical to Glorp's text;
`test_purchases.py` **not**.

| Pillar | Run 2 (old prompts) | Run 4 (new prompts) |
|---|---|---|
| Efficiency | Fail: N+1 in `customer_totals` | **Pass**: measured 1 SQL statement for 50 customers |
| Maintainability | A, MI 65.3, max CC 9 | A, MI 66.7, max CC 9 |
| Security | parameterised; CSV formula cell unescaped | parameterised; **CSV formula cell still unescaped** (`51,"=HYPERLINK(""u"",""v"")",0,0`) |
| Functional | 17/18, 1 test altered by Zed | **17/18, the same test altered the same way** |

- Efficiency: **fixed at the source** by the Glorp prompt, confirmed on the
  exact spec that failed before.
- CSV formula injection: the new rule says to neutralise it "even if the task does
  not mention it"; Glorp did not. The rule exists and was not followed.
- Zed's test mutation is **identical in runs 2 and 4**:
  `"'Quoted' Cafe"` → `'Smith, "Bob"'` at `test_..._percent_and_quote_literally`
  (the file later has a comma+double-quote name test; the model appears to bleed
  that value into the earlier line). The new CONVENTIONS rule was in aider's
  context (log: "Added ../../etc/CONVENTIONS.md to the chat") and **had no effect**.
  It is deterministic, content-triggered, not sampling noise; it did not occur in
  run 3's different test data.

---

## Prompt change 2 (2026-10-05) — after run 4

Run 4 showed Glorp skipping the CSV rule. Added to the same Glorp bullet: a
mandatory **"Code review" section in the dispatch comment**, one line per
applicable rule naming the function and the answer ("`customer_totals: 1 query
(LEFT JOIN + GROUP BY)`"), with "a line you cannot honestly write means the code
is not ready". Output leaving the process as CSV/HTML/SQL always gets a line.
Backup: `~/harness/state/glorp-AGENTS.md.bak-20261005b`. Zed's prompt unchanged.

---

## Run 5 — SOR-117, run 2's spec again (`invoices`, header text now consistent)

Unit SOR-118 `done`; **both files byte-identical to Glorp's text**.

| Pillar | Result | Evidence |
|---|---|---|
| Functional correctness | **Pass** | 19/19 pytest, 100 % line+branch coverage. |
| Maintainability | **Pass** | radon CC avg 3.6 (A), max 5; MI 66.2 (A). |
| Security | **Pass** | Parameterised SQL; `_csv_cell` prefixes `'` to cells starting `= + - @` (verified on `=HYPERLINK(...)`, `+1`, `-2`, `@SUM(A1)`); comma/quote still quoted by `csv`. |
| Efficiency | **Pass** | `LEFT JOIN … GROUP BY`; measured 1 SQL statement for 50 customers. |

Glorp's dispatch comment carried a "Code review" section with one line per
function (query count, bound parameters, LIKE avoided, CSV quoting).

---

## Summary across the five runs

| Run | Prompts | Correct | Maintainable | Secure | Efficient |
|---|---|---|---|---|---|
| 1 notes | original | pass | pass | pass | **fail (N+1)** |
| 2 orders | original | **fail (test altered by Zed)** | pass | gap (CSV formula) | **fail (N+1)** |
| 3 lending | change 1 | pass | pass | pass | pass |
| 4 purchases (= run 2 spec) | change 1 | **fail (same test altered)** | pass | gap (CSV formula) | pass |
| 5 invoices (= run 2 spec) | change 1 + 2 | pass | pass | pass | pass |

### What this does and does not show

- **Efficiency** failed 2/2 on the original prompts and passed 3/3 after the
  change, including an exact-spec A/B (runs 2, 4, 5). The defect was fixed where
  it was written (Glorp's embedded code), not caught afterwards.
- **Security (CSV formula)** needed the second change (explicit self-review lines)
  before Glorp acted on it. One passing run; not shown to be stable.
- **Maintainability** passed every run on every tool; not a problem here.
- **Zed's test mutation** (runs 2 and 4, identical line, identical wrong value)
  was **not** fixed by the CONVENTIONS rule. Absent in runs 3 and 5. It looks
  content-triggered, but the cause is **not established**; I did not test an
  aider edit-format change or a model change.
- **Reviewer agents were not created.** The goal made them conditional on prompt
  changes failing because of the Zed model's limits. The failure that prompts did
  not cure (the test mutation) is the one a real test gate catches, and on Sorbit
  the per-unit mocha gate would turn it into a strike plus a retry with the
  failure appended. Whether that retry actually repairs it is **unproven**: on
  `scratch` there is no gate, so it was never exercised.
- **Samples are small** (5 runs, one repo of three toy modules, Python/SQLite).
  Nothing here was run on Sorbit or on JS/Dart, which is where the harness's real
  work is. These are measurements on one problem family, not a guarantee.

### Findings about the harness itself (not about the code)

1. **`scratch` has no manifest, so the per-unit gate and the verify gate never
   run there.** Units report `job succeeded` with a failing test (run 2/4), and
   every Verify issue went red with "no suite: neither .suite nor a manifest tests
   field". `goal-fanout` does not pass `Tests:` through. Evaluating the
   functional pillar through the harness's own gate needs a `scratch.json`
   manifest with `tests`; not added, because it changes live Z440 config and
   FANOUT.md uses manifest-less `scratch` as the regression guard.
2. A manual `POST /agents/<verity>/wakeup` is a no-op for a stuck verify issue
   (ONBOARDING §13.5 reproduced): the run "succeeded" in 1.6 s and wrote nothing.
3. Goals SOR-105/108/111/114/117 are parked `blocked` on Verify issues
   (SOR-107/110/113/116/119, `todo`) for a human. Branches `agent/goal-{notes-store,
   orders-module,lending-module,purchases-module,invoices-module}` on the Z440's
   `scratch` are unmerged test artifacts; nothing was published to GitHub.

---

## Manifest added to `scratch`, then runs 6 and 7 (2026-10-05)

Requested after the five runs above: close the gate gap.

**Change:** new file `/home/agent/etc/repos/scratch.json` (no backup: nothing
existed before) with `tests` =
`PYTHONDONTWRITEBYTECODE=1 python3 -m pytest -q -p no:cacheprovider` and no
`base`, so base resolution is unchanged (HEAD = `main`). `-p no:cacheprovider`
and the env var keep pytest from writing cache/bytecode files into worktrees.
Board was idle first (no `in_progress`, `work/`+`jobs/` empty, GPU lock free).

**Proof the gate works, before spending a GPU run:** `verifyd` directly on two
existing branches. `agent/goal-invoices-module` → **exit 0, 20 passed**.
`agent/goal-purchases-module` (run 4, Zed's altered test) → **exit 1,
`FAILED test_purchases.py::test_find_customers_treats_percent_and_quote_literally`,
1 failed, 18 passed**. That branch used to report "no suite". No worktree left
in `/home/agent/verify`.

FANOUT.md's note that manifest-less `scratch` is the regression guard for "no
manifest = pre-manifest behaviour" no longer holds for `scratch`; that guard now
needs a different throwaway repo.

### Run 6 — SOR-120, run 2's spec again (`payments`)

Unit SOR-122 `done` on the **first attempt** (no strike), verify SOR-123
**green** (real pytest suite), goal `in_review`. Both files byte-identical to
Glorp's text.

| Pillar | Result | Evidence |
|---|---|---|
| Functional correctness | **Pass** | 19/19; 100 % line+branch; verify green through the gate. |
| Maintainability | **Pass** | CC avg 3.6 (A), MI 66.2 (A), lizard clean. |
| Security | **Pass** | Parameterised; `=HYPERLINK(...)`, `+1`, `@SUM(A1)` come out `'`-prefixed; `%` and `x' OR '1'='1` match nothing. |
| Efficiency | **Pass** | 1 SQL statement for 54 customers. |

### Run 7 — SOR-121, run 1's spec again (`journal`)

Unit SOR-124 `done` first attempt, verify SOR-125 **green**, goal `in_review`.
Both files byte-identical to Glorp's text.

| Pillar | Result | Evidence |
|---|---|---|
| Functional correctness | **Pass** | 18/18; verify green. 92 % coverage: the uncovered lines are `add_entry`'s `except: rollback; raise` (the atomic-write rule is followed, its failure path is not tested). |
| Maintainability | **Pass** | CC avg 5.0 (A), MI 71.4 (A), lizard clean. |
| Security | **Pass** | Parameterised; `%` and `'; DROP TABLE entries;--` match nothing, table intact. |
| Efficiency | **Pass** | `entries_with_tags` is one `LEFT JOIN`; 1 SQL statement for 50 entries (run 1 issued 51). |

### What these two runs do and do not show

- With the gate live, the harness delivered two goals end to end (unit gate →
  fan-in → verify green → `in_review`), and the code passed all four pillars
  without any reviewer agent.
- Run 1's spec failed efficiency on the original prompts; the same spec on the
  current prompts passes. Run 2's spec failed efficiency and correctness; it
  now passes on three consecutive runs (5, 6, and run 4's efficiency).
- **The retry path is still unexercised.** Zed's test mutation did not recur in
  runs 5, 6 or 7, so no unit failed its gate, no strike was recorded and no
  retry ran. Whether a gated retry repairs that mutation remains **unproven**.
  The gate's ability to *detect* the mutation is proven (above); its ability to
  *recover* is not.
- Still 7 toy Python/SQLite modules on `scratch`; nothing on Sorbit.
- Not changed: no prompt edits in this round, no reviewer agents.

---

## Forced-failure probes of the retry path (2026-10-05)

Question left open above: does the strike → retry → Sentinel path actually work,
and does a retry repair a failing unit? Glorp was bypassed: hand-written plans
through `~/harness/bin/goal-fanout` (as `charlie`; run without
`PAPERCLIP_API_KEY`, so the goal issue was not parked on verify, which does not
matter here), against `scratch` with the new manifest. Goal issues SOR-126 and
SOR-129 are test scaffolding, not real goals.

### Probe 1 — SOR-127: prose contradicts the embedded test

Test asserts hyphens; prose said "replace whitespace with an underscore".
**Did not fail.** Zed read the exact test, wrote hyphens and rewrote the
docstring to say hyphen. First attempt `exit 0`. Zed reconciled the conflict on
its own, in favour of the tests. Good behaviour, but no retry was exercised.

### Probe 2 — SOR-130: unit names a third file the task never mentions

`Files: wordcount.py test_wordcount.py WORDCOUNT_NOTES.md`; the task text only
describes the first two. Expected: under-applied gate (exit 6).

| Step | What happened |
|---|---|
| Attempt 1 | exit 6. pytest passed (4); `UNDER-APPLIED … Not changed at all: WORDCOUNT_NOTES.md`. Branch not advanced. |
| Retry monitor | Scheduled ~2 s out each time; attempts 2 and 3 followed within ~1 min. |
| Attempts 2, 3 | Identical: exit 6, same file. The feedback named the file but Zed had no content for it and left it empty. |
| Strike 3 | Escalation comment posted, unit handed to **Sentinel**. |
| Sentinel | Correct diagnosis (task text never described the third file). Rewrote the task to give all three files' full contents in fenced blocks, "exactly this"; wrote a retry-cycle-reset comment; returned the unit to Zed. It cited SOR-89 as the same failure shape. |
| Attempt 4 | `exit 0`, 3 files changed, no test mutation. Fan-in, **verify green** (4 passed), unit and verify `done`, goal `in_review`. |

So: detection, strike counting, auto-retry, escalation, rewrite, reset and
recovery **all ran end to end on a real failure**. The path works.

### What the probe showed beyond "it works"

1. **Plain retry did not repair this failure; Sentinel did.** The feedback appended
   to retries named the missing file, but a model cannot invent content for a file
   the task does not describe. Three identical GPU attempts were wasted, which is
   the designed cost bound (3), not a bug. The recovery came from the task rewrite,
   so for failures caused by the *task text* the load-bearing component is
   Sentinel, not retry.
2. **Sentinel also found a second defect I did not know about.** Attempt 1's
   embedded test had a real line break inside a string literal (`"\n"` became a
   raw newline), a SyntaxError that aider repaired itself with a silent edit to the
   test text. I reproduced a cause: `paperclipai issue create --description`
   turns a literal `\n` into a real newline (`\t` survives). That is harness-side
   text mangling, not bad planning.
   **Correction to run 1:** I attributed the raw newline in `"\t\n"` to
   Glorp's embedded text. It more likely came from this conversion on the way into
   the issue description. I did not trace which exact call converted it on the
   `child:create --payload-json` path that `goal-fanout` really uses, so the
   precise point is **unconfirmed**.
3. Sentinel's rewrite sidestepped it by writing the assertion with `chr(9)` and
   `chr(10)`, which is a workaround for an escape bug, not a fix of it.
4. The Zed test mutation seen in runs 2 and 4 is still unexplained and did not
   occur here. It is **not** shown to be repaired by a retry, because it did not
   happen in this probe. What is now proven is that *a* failing unit recovers
   through retry+Sentinel; the mutation case specifically is still untested.

### Residue

Three worktrees were left in `/home/agent/work` (`20261005-150835-14270`,
`…150905-30409`, `…150945-27142`): jobd keeps a worktree when it has
uncommitted work ("kept, uncommitted work present"), here the empty untracked
`WORDCOUNT_NOTES.md`. Not removed. `agent-prune` clears worktrees older than 7
days. Also SOR-132 (escape test) cancelled. Branches `agent/goal-retry-probe` and
`agent/goal-retry-probe-2` are unmerged test artifacts.

---

## Appendix A — Run 1 output files

### notes.py

```python
"""A small SQLite-backed notes store with tags. Standard library only."""


def init_db(conn):
    """Create the notes and tags tables. Safe to call more than once."""
    conn.execute(
        """
        CREATE TABLE IF NOT EXISTS notes (
            id    INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            body  TEXT NOT NULL DEFAULT ''
        )
        """
    )
    conn.execute(
        """
        CREATE TABLE IF NOT EXISTS tags (
            note_id INTEGER NOT NULL REFERENCES notes(id) ON DELETE CASCADE,
            tag     TEXT    NOT NULL,
            PRIMARY KEY (note_id, tag)
        )
        """
    )
    conn.commit()


def add_note(conn, title, body, tags=()):
    """Store a note and its tags; return the new note id.

    The title is stripped of surrounding whitespace and must then be a
    non-empty string of at most 200 characters, otherwise ValueError.
    Tags are lowercased and de-duplicated.
    """
    if not isinstance(title, str):
        raise ValueError("title must be a string")
    clean = title.strip()
    if not clean or len(clean) > 200:
        raise ValueError("title must be 1-200 characters after stripping")
    cur = conn.execute("INSERT INTO notes (title, body) VALUES (?, ?)", (clean, body))
    note_id = cur.lastrowid
    for tag in dict.fromkeys(t.lower() for t in tags):
        conn.execute("INSERT OR IGNORE INTO tags (note_id, tag) VALUES (?, ?)", (note_id, tag))
    conn.commit()
    return note_id


def search_notes(conn, term):
    """Ids of notes whose title or body contains term, case-insensitively, by id.

    An empty term returns an empty list. The term is matched literally, so
    characters such as % and _ are not treated as wildcards.
    """
    if not term:
        return []
    needle = term.lower()
    hits = []
    for note_id, title, body in conn.execute("SELECT id, title, body FROM notes ORDER BY id"):
        if needle in title.lower() or needle in body.lower():
            hits.append(note_id)
    return hits


def notes_with_tags(conn):
    """Every note as {"id", "title", "body", "tags"}, ordered by id.

    Each tags value is a sorted list of strings.
    """
    out = []
    for note_id, title, body in conn.execute("SELECT id, title, body FROM notes ORDER BY id"):
        rows = conn.execute("SELECT tag FROM tags WHERE note_id = ?", (note_id,))
        out.append({
            "id": note_id,
            "title": title,
            "body": body,
            "tags": sorted(row[0] for row in rows),
        })
    return out
```

### test_notes.py

```python
"""Tests for notes.py. Run with: python3 -m pytest -q"""

import sqlite3

import notes


def fresh():
    conn = sqlite3.connect(":memory:")
    notes.init_db(conn)
    return conn


def test_init_db_is_safe_to_call_twice():
    conn = fresh()
    notes.init_db(conn)
    notes.init_db(conn)
    assert notes.notes_with_tags(conn) == []


def test_empty_database():
    conn = fresh()
    assert notes.search_notes(conn, "anything") == []
    assert notes.notes_with_tags(conn) == []


def test_add_and_read_back_a_note():
    conn = fresh()
    note_id = notes.add_note(conn, "Shopping", "milk and eggs", tags=["home", "food"])
    assert note_id == 1
    assert notes.notes_with_tags(conn) == [
        {"id": 1, "title": "Shopping", "body": "milk and eggs", "tags": ["food", "home"]}
    ]


def test_title_is_stripped_before_storing():
    conn = fresh()
    note_id = notes.add_note(conn, "   Padded   ", "body")
    assert notes.notes_with_tags(conn)[0]["title"] == "Padded"
    assert note_id == 1


def test_title_of_exactly_200_characters_is_allowed():
    conn = fresh()
    notes.add_note(conn, "x" * 200, "body")
    assert notes.notes_with_tags(conn)[0]["title"] == "x" * 200


def test_empty_title_is_rejected():
    conn = fresh()
    for bad in ["", "   ", "\t\n"]:
        try:
            notes.add_note(conn, bad, "body")
        except ValueError:
            pass
        else:
            raise AssertionError("ValueError not raised for title %r" % (bad,))
    assert notes.notes_with_tags(conn) == []


def test_over_long_title_is_rejected():
    conn = fresh()
    try:
        notes.add_note(conn, "y" * 201, "body")
    except ValueError:
        pass
    else:
        raise AssertionError("ValueError not raised for a 201-character title")
    assert notes.notes_with_tags(conn) == []


def test_duplicate_tags_are_deduplicated_and_lowercased():
    conn = fresh()
    notes.add_note(conn, "Recipe", "body", tags=["Soup", "soup", "SOUP", "dinner"])
    assert notes.notes_with_tags(conn)[0]["tags"] == ["dinner", "soup"]


def test_search_matches_title_or_body_case_insensitively():
    conn = fresh()
    a = notes.add_note(conn, "Alpha", "first note")
    b = notes.add_note(conn, "Beta", "mentions ALPHA in the body")
    c = notes.add_note(conn, "Gamma", "unrelated")
    assert notes.search_notes(conn, "alpha") == [a, b]
    assert notes.search_notes(conn, "gamma") == [c]
    assert notes.search_notes(conn, "nothing here") == []


def test_search_with_empty_term_returns_empty_list():
    conn = fresh()
    notes.add_note(conn, "Alpha", "body")
    assert notes.search_notes(conn, "") == []


def test_search_treats_percent_and_quote_literally():
    conn = fresh()
    a = notes.add_note(conn, "50% off", "sale")
    b = notes.add_note(conn, "no percent", "it's fine")
    c = notes.add_note(conn, "plain", "nothing special")
    assert notes.search_notes(conn, "50%") == [a]
    assert notes.search_notes(conn, "%") == [a]
    assert notes.search_notes(conn, "'") == [b]
    assert notes.search_notes(conn, "'%'") == []


def test_search_underscore_is_literal():
    conn = fresh()
    a = notes.add_note(conn, "snake_case", "body")
    b = notes.add_note(conn, "snakeXcase", "body")
    assert notes.search_notes(conn, "_") == [a]


def test_unicode_text_round_trips_and_searches():
    conn = fresh()
    a = notes.add_note(conn, "Café Paris", "naïve crème brûlée")
    b = notes.add_note(conn, "Tōkyō", "東京のメモ")
    assert notes.notes_with_tags(conn)[0]["title"] == "Café Paris"
    assert notes.notes_with_tags(conn)[1]["body"] == "東京のメモ"
    assert notes.search_notes(conn, "café") == [a]
    assert notes.search_notes(conn, "café".upper()) == [a]
    assert notes.search_notes(conn, "東京") == [b]


def test_notes_with_tags_orders_by_id_and_sorts_tags():
    conn = fresh()
    notes.add_note(conn, "First", "one", tags=["zulu", "alpha"])
    notes.add_note(conn, "Second", "two")
    notes.add_note(conn, "Third", "three", tags=["mike"])
    assert notes.notes_with_tags(conn) == [
        {"id": 1, "title": "First", "body": "one", "tags": ["alpha", "zulu"]},
        {"id": 2, "title": "Second", "body": "two", "tags": []},
        {"id": 3, "title": "Third", "body": "three", "tags": ["mike"]},
    ]
```

---

## Appendix B — Run 2 output (orders.py)

```python
"""A small SQLite-backed customer orders store. Standard library only."""

import csv
import io


def init_db(conn):
    """Create the customers and orders tables. Safe to call more than once."""
    conn.execute(
        """
        CREATE TABLE IF NOT EXISTS customers (
            id    INTEGER PRIMARY KEY AUTOINCREMENT,
            name  TEXT NOT NULL,
            email TEXT NOT NULL
        )
        """
    )
    conn.execute(
        """
        CREATE TABLE IF NOT EXISTS orders (
            id           INTEGER PRIMARY KEY AUTOINCREMENT,
            customer_id  INTEGER NOT NULL REFERENCES customers(id),
            amount_cents INTEGER NOT NULL
        )
        """
    )
    conn.commit()


def add_customer(conn, name, email):
    """Store a customer and return the new id.

    The name is stripped of surrounding whitespace and must then be a
    non-empty string of at most 100 characters. The email must be a string
    containing exactly one "@" with text on both sides. Emails are unique
    when compared case-insensitively; a duplicate raises ValueError.
    """
    if not isinstance(name, str):
        raise ValueError("name must be a string")
    clean = name.strip()
    if not clean or len(clean) > 100:
        raise ValueError("name must be 1-100 characters after stripping")
    if not isinstance(email, str):
        raise ValueError("email must be a string")
    mail = email.strip()
    parts = mail.split("@")
    if len(parts) != 2 or not parts[0] or not parts[1]:
        raise ValueError('email must contain exactly one "@" with text on both sides')
    existing = conn.execute(
        "SELECT id FROM customers WHERE lower(email) = ?", (mail.lower(),)
    ).fetchone()
    if existing is not None:
        raise ValueError("email is already registered")
    cur = conn.execute(
        "INSERT INTO customers (name, email) VALUES (?, ?)", (clean, mail)
    )
    conn.commit()
    return cur.lastrowid


def add_order(conn, customer_id, amount_cents):
    """Store an order and return its new id.

    amount_cents must be a positive int (bool is not accepted) and the
    customer must exist, otherwise ValueError.
    """
    if isinstance(amount_cents, bool) or not isinstance(amount_cents, int):
        raise ValueError("amount_cents must be an int (bool is not accepted)")
    if amount_cents <= 0:
        raise ValueError("amount_cents must be positive")
    known = conn.execute(
        "SELECT id FROM customers WHERE id = ?", (customer_id,)
    ).fetchone()
    if known is None:
        raise ValueError("unknown customer_id")
    cur = conn.execute(
        "INSERT INTO orders (customer_id, amount_cents) VALUES (?, ?)",
        (customer_id, amount_cents),
    )
    conn.commit()
    return cur.lastrowid


def find_customers(conn, prefix):
    """Ids of customers whose name starts with prefix, case-insensitively, by id.

    An empty prefix returns an empty list. The prefix is matched literally,
    so characters such as % and _ are not treated as wildcards.
    """
    if not prefix:
        return []
    needle = prefix.lower()
    return [
        row[0]
        for row in conn.execute("SELECT id, name FROM customers ORDER BY id")
        if row[1].lower().startswith(needle)
    ]


def customer_totals(conn):
    """Every customer as {"id", "name", "orders", "total_cents"}.

    Customers with no orders are included with 0 orders and 0 cents. The
    result is ordered by total_cents descending, then by id.
    """
    out = []
    for customer_id, name in conn.execute("SELECT id, name FROM customers ORDER BY id"):
        row = conn.execute(
            "SELECT COUNT(*), COALESCE(SUM(amount_cents), 0) "
            "FROM orders WHERE customer_id = ?",
            (customer_id,),
        ).fetchone()
        out.append({
            "id": customer_id,
            "name": name,
            "orders": row[0],
            "total_cents": row[1],
        })
    out.sort(key=lambda d: (-d["total_cents"], d["id"]))
    return out


def export_csv(conn):
    """The customer_totals as CSV text, with a header row id,name,orders,total_cents."""
    buf = io.StringIO()
    writer = csv.writer(buf)
    writer.writerow(["id", "name", "orders", "total_cents"])
    for row in customer_totals(conn):
        writer.writerow([row["id"], row["name"], row["orders"], row["total_cents"]])
    return buf.getvalue()
```

---

## Appendix C — Run 5 output (invoices.py)

```python
"""A small SQLite-backed customer invoices store. Standard library only."""

import csv
import io


def init_db(conn):
    """Create the customers and invoices tables. Safe to call more than once."""
    conn.execute(
        """
        CREATE TABLE IF NOT EXISTS customers (
            id    INTEGER PRIMARY KEY AUTOINCREMENT,
            name  TEXT NOT NULL,
            email TEXT NOT NULL
        )
        """
    )
    conn.execute(
        """
        CREATE TABLE IF NOT EXISTS invoices (
            id           INTEGER PRIMARY KEY AUTOINCREMENT,
            customer_id  INTEGER NOT NULL REFERENCES customers(id),
            amount_cents INTEGER NOT NULL
        )
        """
    )
    conn.commit()


def add_customer(conn, name, email):
    """Store a customer and return the new id.

    The name is stripped of surrounding whitespace and must then be a
    non-empty string of at most 100 characters. The email must be a string
    containing exactly one "@" with text on both sides. Emails are unique
    when compared case-insensitively; a duplicate raises ValueError.
    """
    if not isinstance(name, str):
        raise ValueError("name must be a string")
    clean = name.strip()
    if not clean or len(clean) > 100:
        raise ValueError("name must be 1-100 characters after stripping")
    if not isinstance(email, str):
        raise ValueError("email must be a string")
    mail = email.strip()
    parts = mail.split("@")
    if len(parts) != 2 or not parts[0] or not parts[1]:
        raise ValueError('email must contain exactly one "@" with text on both sides')
    existing = conn.execute(
        "SELECT id FROM customers WHERE lower(email) = lower(?)", (mail,)
    ).fetchone()
    if existing is not None:
        raise ValueError("email is already registered")
    cur = conn.execute(
        "INSERT INTO customers (name, email) VALUES (?, ?)", (clean, mail)
    )
    conn.commit()
    return cur.lastrowid


def add_invoice(conn, customer_id, amount_cents):
    """Store an invoice and return its new id.

    amount_cents must be a positive int (bool is not accepted) and the
    customer must exist, otherwise ValueError.
    """
    if isinstance(amount_cents, bool) or not isinstance(amount_cents, int):
        raise ValueError("amount_cents must be an int (bool is not accepted)")
    if amount_cents <= 0:
        raise ValueError("amount_cents must be positive")
    known = conn.execute(
        "SELECT id FROM customers WHERE id = ?", (customer_id,)
    ).fetchone()
    if known is None:
        raise ValueError("unknown customer_id")
    cur = conn.execute(
        "INSERT INTO invoices (customer_id, amount_cents) VALUES (?, ?)",
        (customer_id, amount_cents),
    )
    conn.commit()
    return cur.lastrowid


def find_customers(conn, prefix):
    """Ids of customers whose name starts with prefix, case-insensitively, by id.

    An empty prefix returns an empty list. The leading characters of the name
    are compared for equality rather than with LIKE, so characters such as %
    and _ are matched literally and need no escaping.
    """
    if not prefix:
        return []
    return [
        row[0]
        for row in conn.execute(
            "SELECT id FROM customers "
            "WHERE lower(substr(name, 1, ?)) = lower(?) "
            "ORDER BY id",
            (len(prefix), prefix),
        )
    ]


def _csv_cell(value):
    """Neutralise a text cell a spreadsheet would otherwise run as a formula.

    Stored names are stripped, so they cannot begin with a tab or a carriage
    return; only =, +, - and @ can start one. Prefixing an apostrophe is the
    usual way to make such a cell inert while leaving the text readable.
    """
    if isinstance(value, str) and value[:1] in ("=", "+", "-", "@"):
        return "'" + value
    return value


def customer_totals(conn):
    """Every customer as {"id", "name", "invoices", "total_cents"}.

    Customers with no invoices are included with 0 invoices and 0 cents.
    The result is ordered by total_cents descending, then by id.
    """
    rows = conn.execute(
        "SELECT c.id, c.name, COUNT(i.id), COALESCE(SUM(i.amount_cents), 0) "
        "FROM customers AS c "
        "LEFT JOIN invoices AS i ON i.customer_id = c.id "
        "GROUP BY c.id, c.name "
        "ORDER BY COALESCE(SUM(i.amount_cents), 0) DESC, c.id"
    ).fetchall()
    return [
        {
            "id": row[0],
            "name": row[1],
            "invoices": row[2],
            "total_cents": row[3],
        }
        for row in rows
    ]


def export_csv(conn):
    """The customer_totals as CSV text, with a header row id,name,invoices,total_cents."""
    buf = io.StringIO()
    writer = csv.writer(buf)
    writer.writerow(["id", "name", "invoices", "total_cents"])
    for row in customer_totals(conn):
        writer.writerow(
            [row["id"], _csv_cell(row["name"]), row["invoices"], row["total_cents"]]
        )
    return buf.getvalue()
```
