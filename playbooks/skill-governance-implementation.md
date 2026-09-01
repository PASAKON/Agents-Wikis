# Skill governance — DEV implementation spec (ADR 0022)

**Companion to:** [ADR 0022](../decisions/0022-skill-governance-visibility-authorship-audit.md).
The ADR carries the reasoning; this page carries the work.

**Read before starting:** ADR 0022 §9 lists three items awaiting CEO sign-off.
Wave 0 and Wave 1 do not depend on any of them. Wave 2 depends on item 1.

**Repo:** `/Users/gob/Projects/Agents`. On Contabo the same repo is at
`/opt/mooniex-agents` — **never hardcode either path**; derive the root, and test
that you did.

## Standing constraints for every wave

- **Tests go in `scripts/test_*.py` or `lib/test_*.py`.** `pytest.ini` sets
  `testpaths = scripts lib`. A `tests/` directory does not exist and pytest will
  never look there.
- **`python3` on PATH has no PyYAML** — measured, both `/opt/homebrew/bin/python3`
  and `/usr/bin/python3` raise `ModuleNotFoundError: No module named 'yaml'`. Only
  `.venv` has it. **Anything invoked as a bare `python3` hook must not import yaml.**
- Run `pytest` before submitting. 770 tests pass today; keep it green.
- Deliverables that must survive review go in `docs/`, never `state/` — merge
  cleans the worktree and `state/` is gitignored.
- Never rename a directory under `.claude/skills/` in these waves. See Wave 2.

---

## Wave 0 — stop the bleeding (do this first, independent of everything else)

Three live defects. None is new work; all three are losses happening now.

### 0.1 Worker skill telemetry is being destroyed

`scripts/hook-skill-log.py:20` sets
`LOG_PATH = Path(__file__).resolve().parent.parent / "state" / "skill-usage.log"`.
From a worktree, `__file__` is the worktree's own copy, so fires land in
`worktrees/<...>/state/skill-usage.log` and die when the worktree is reaped.
Cross-repo worktrees never run the hook at all — those repos have no
`.claude/settings.json`.

**Fix:** register the `PostToolUse` Skill hook **once in `~/.claude/settings.json`**
with an absolute script path and an absolute default log path.

This is a user-settings edit, which the classifier blocks for agents — **prepare
the exact JSON and hand it to the CEO to paste.** Do not attempt to write it.

Do NOT implement git-common-dir discovery. It was measured to resolve to the
*wrong* repo from a cross-repo worktree
(`worktrees/comfy-runpod-worker__browser_operator__task-610ce4dc` →
`/Users/gob/Projects/comfy-runpod-worker/.git`).

**Acceptance:** a skill fired from inside a worktree appends to the main repo's
`state/skill-usage.log`. Verify by firing one and reading the file.

### 0.2 Recover the 79 stranded fires

One shell line, run once, then never again:

```
cat worktrees/*/state/skill-usage.log >> state/skill-usage.log \
  && sort -u -o state/skill-usage.log state/skill-usage.log
```

The stranded fires are only three skills — browser-operator 38,
higgsfield-unlimited-gen 36, gdrive-filing 5 — all already top-ranked, so this
changes no decision. Do it for completeness, do not build a script for it.

Note `state/skill-usage.log` is gitignored at `.gitignore:20`. That is deliberate;
leave it.

### 0.3 Add the role column

`scripts/hook-skill-log.py` grows from 3 fields to 4:
`ts \t skill \t session \t role`, role from `WORKER_ROLE`, else `-`.
Legacy 3-field lines pad on read.

Export `WORKER_ROLE` where the worker is spawned. **Without this column, Wave 3
Phase 2 cannot be evidence-based** — it is the whole reason Wave 3 is phased.

**Acceptance:** a worker fire logs its role; a C-level fire logs `-`; the existing
422 lines still parse.

---

## Wave 1 — provenance: the field is the authority

### 1.1 Frontmatter contract

Add to org-authored skills (see ADR 0022 §3 for the full block):

| Key | Values | Rule |
|---|---|---|
| `audience` | list of role keys, or `all` / `cxo` / `worker` | the routing key |
| `created_by` | **exactly** `human` or `agent` | anything else silently coerces to `human` at `skill-curator.py:157` |
| `author` | `{role, date}` | the rich stamp. **The role goes here, never in `created_by`** |
| `improved_by` | list, optional, append-only | CEO rule 8 |
| `aka` | list, optional | preserves telemetry across a future rename |

### 1.2 `scripts/skill-lint.py` (new, ~80 lines)

Verbs `check` and `--json`. Five codes: missing SKILL.md · unparseable frontmatter ·
`name` ≠ directory basename · `created_by` outside the enum · `audience` token not
a known role or group.

Reuse `skill-curator.py:_resolve_within()` for the symlink refusal rather than
reimplementing it. Group tokens come from `agents()['roles']` levels, not a new table.

Also emit a **loud warning** when `created_by` is coerced — today that failure is
silent, which is how a skill leaves the curator's reach without anyone noticing.

Wire into the **existing** `.git/hooks/pre-commit`: three lines, run the lint when
staged paths touch `.claude/skills/`. Skills are committed files, so this is
complete coverage with no settings paste.

**This is a lint, not a gate.** It must not block authoring — CEO decisions 4 and 10.

### 1.3 Backfill

The 21 org skills missing `created_by` get `created_by: human` and `audience:`.
**Lazy for `author:`** — a skill gains it the next time someone edits it; until
then objections route to CTO, which is where they were going anyway.

Do not bulk-rewrite skill bodies in this wave.

Test: `scripts/test_skill_lint.py`.

---

## Wave 2 — git is the ledger (needs CEO sign-off item 1 only for the prefix half)

Nothing is built from scratch here. See ADR 0022 §4 for why the bespoke ledger and
blob store were cancelled.

### 2.1 Curator commits its own mutations

Each mutating verb (`archive`, `restore`, `pin`, `unpin`, plus the new `create`
that reverses ADR 0018 §6) ends with `git add` on the touched skill dirs and
`git commit` carrying a trailer:

```
Skill-Actor: <role>/<session-id>
```

Identity from `tools.agent_transport`. Roughly 20 lines.

### 2.2 Three read verbs (~35 lines total)

| Verb | Implementation |
|---|---|
| `history [--skill NAME]` | `git log --follow -- .claude/skills/<name>` |
| `undo <sha>` | `git revert` |
| `drift` | `git status --porcelain -- .claude/skills` |

### 2.3 Deletions

- Delete `state/skill-usage.json` handling: `_load_state`, `_save_state`, and the
  `state_path` field. **The file has never been written**, so there is nothing to
  migrate. Lifecycle state (`pinned`, `lifecycle`, `archived_at`) moves into each
  SKILL.md's frontmatter.
- Beware `skill-curator.py:306` — `names |= set(existing_state.keys())`. Any
  non-skill key placed in that state would enumerate as a phantom skill. Removing
  the sidecar removes that hazard; do not reintroduce it elsewhere.
- `_backup_skill()`'s `copytree` into `state/skill-curator-backups/` is now
  redundant with git. Leave it in place this wave; retire it only after `undo` is
  proven in use.

### 2.4 Prefix — new skills only

`skill-author` names every **new** skill with its audience prefix. The existing 21
directories are **grandfathered permanently**.

**Do not `git mv` any skill directory.** Measured: `~/.claude/skills` contains
symlinks into this repo (`browser-operator`, `mooniex-finance`, ...). Renaming the
target dangles the link and the skill **vanishes with no error**. 39 files also
name skills as literal strings, including prompt text in `runners/secretary_server.py`.

Reports render `cto / merge-checklist` computed from `audience`, which is what the
prefix is for.

### 2.5 `skill-author` loses its global write path

Remove `~/.claude/skills/<name>/` as a sanctioned location for org-authored skills.
A skill written there sits outside git, outside the curator and outside `undo` by
construction.

---

## Wave 3 — visibility (Phase 1 ships now, Phase 2 waits two weeks)

Mechanism and measurements: ADR 0022 §2. It is **proven on Claude Code 2.1.252**,
not assumed — re-verify on the installed version before shipping, and make the
version check a test, because this is the one part an upgrade can silently break.

### ⚠️ Measured outcome of Phase 1 (2026-09-01) — read before planning Phase 2

`ecc@ecc` **cannot be disabled.** GateGuard's `gateguard-fact-force` hook is
registered inside the ecc plugin's own `hooks/hooks.json` bundle, so
`enabledPlugins: false` kills the hook along with the skills. Proven by live A/B
with a fresh `claude` subprocess each way, not by inspection. It is locked in by
`scripts/test_skill_visibility.py::test_worker_baseline_never_disables_gateguard_bundle`.

ecc owns **92 of the 113 plugin skills**, so the shipped profile disables three
plugins, not four, and the real saving is:

| Config | Skills | Tokens |
|---|---|---|
| Baseline | 186 | 39,993 |
| All 4 plugins off (unsafe — GateGuard dies) | 76 | 34,915 |
| **Shipped (3 off, ecc kept)** | **168** | **39,924** |

**−9.7% skills, −0.2% tokens.** ADR 0022 §2's −38.6% projection assumed ecc could
go; it cannot.

**This also caps Phase 2.** Plugin skills bypass `skillOverrides` entirely, so
ecc's 92 skills cannot be hidden by that lever either. The only route to the big
number is **extracting GateGuard out of the ecc bundle into our own hooks**, after
which ecc can be disabled wholesale. Treat that as the real prize; Phase 2's
per-skill lists are worth much less than the ADR implied until it is done.

### Phase 1 — plugins only

- `policies/skill-visibility/worker-baseline.json`, checked in:
  `{"enabledPlugins": {"<plugin>@<marketplace>": false, ...}}`.
  **Read plugin keys verbatim from `~/.claude/plugins/installed_plugins.json`.
  Never synthesize them.**
- `policies/agents.yaml`: one new key per role,
  `skills_profile: worker-baseline | browser | all`. `cto` gets `all`.
- `runners/worker_init.py`: when the profile is not `all`, append
  `--settings <abs path>` to the argv. When it is `all`, **emit nothing and pass
  no flag** — that is how CEO decision 2 costs zero code.
- Same commit: add `Skill` to `_BASE_WORKER_TOOLS`. Today only `web_designer` and
  `browser_operator` get it, so every other role would have visibility configured
  and no tool to use it.

**Check first, and it may save the whole flag:** measure whether `skillOverrides`
is honoured in the `settings.local.json` that `_write_dev_settings()` already
writes at `runners/worker_init.py:176`. If yes, put the keys there and skip
`--settings` entirely. If no, use the flag — note `flagSettings` outranks
`localSettings`, which is the property that stops a worker lifting its own cage.

Keep the generated file **small**. A ~46-key file behaved correctly in
measurement; a 74KB / 2247-key blob behaved anomalously and was not diagnosed.

### Phase 2 — per-skill allow-lists (hold two weeks)

Requires Wave 0.3's role column to have been collecting. Then add
`skillOverrides` maps to two or three profile files (`browser.json`,
`engineering.json`), seeded from **measured** role behaviour.

Self-hiding (`requires_tools` / `requires_roles` in frontmatter) is evaluated
against `worker_tool_grants(role)` and may only **subtract**, never grant.

**Known gap to close before Phase 2:** `scripts/skill-report.py:41-58` cannot
enumerate the full universe (it scans marketplace dirs and prefixes with
`dir.split("@")[0]`), so a generator subtracting from it would under-deny.
Fix discovery first or the allow-lists leak.

---

## Wave 4 — audit and objections

### 4.1 `scripts/skill-report.py` — three columns

| Column | Source |
|---|---|
| `/wk` | fires per week since **that skill's own first fire**. This is the "how often" the CEO asked for; Hermes cannot compute it at all because it stores only totals |
| `updated` | `git -C <repo> log -1 --format=%cI -- SKILL.md` — verified to work through the external symlinks |
| `obj` | open / total objections, from 4.2 |

Add `--brief`, which prints **zero bytes when there is nothing to report**. That
is the one line `/session-open` shows (CEO proposal C, folded in).

Also fold worktree logs into `_load_log()` so nothing is missed if 0.1 regresses.

**Rule, adopted from Hermes' own curator and non-negotiable:** *"'use=0' is not
evidence a skill is valuable; it's absence of evidence either way."* Silence alone
must never archive a skill. Encode it as a test.

### 4.2 `tools/skill_objection.py` (new, ~80 lines)

- `raise_objection(skill, conflict, did_instead, blocked=False)` — resolve the
  author from frontmatter (absent → `cto`), write **one `skill_objection` event**
  through the existing `db.log_event` (no DDL, do not change `lastrowid`
  behaviour), and add a **LungNote to-do** addressed to that role.
- `summary(days=30)` — per-skill count plus the last conflict text, for the `obj`
  column.
- CLI: `raise`, `list`, `summary`. **No `resolve` verb** — completing the LungNote
  to-do is the resolution.
- One MCP tool on `runners/worker_mcp_server.py` delegating to it, plus the
  matching `_BASE_WORKER_TOOLS` grant in the same commit.

Test `scripts/test_skill_objection.py`: event written · role resolved from
frontmatter · absent author → cto. Monkeypatch `db.DB_PATH` the way
`scripts/test_send_to_cto.py` does.

### 4.3 The part that decides whether any of this gets used

`skill-author` must require an **"If this skill is wrong"** section in every skill
body, naming the objection verb. A channel nobody is told about gets zero traffic.
One paragraph.

### 4.4 Task-outcome smoke (optional, verify before relying on it)

A task that fired skill X and ended `failed` / `stalled` is smoke against X.
**Verify the join first**: does a session id in `state/skill-usage.log` actually
match anything in `state/tasks.db`? If it does not, say so and stop — do not
report an unjoined correlation as a defect rate.

---

## Wave 5 — authoring doctrine

Full reasoning in ADR 0022 §7. The contract is **binary**:

> A rule block is either **HARD**, and carries a **`Why hard:`** clause, or it is
> advice the model may override — and when it overrides, it says why.

### 5.1 `.claude/skills/skill-author/SKILL.md`

This is where behaviour actually changes. Replace the `**Operating rules** —
bullet list of guardrails (Never X, Always Y)` body item with `**Rules, tiered**`,
add the HARD test, and tier `skill-author`'s own operating rules in the same commit.

Keep FACT and WHY as **prose advice — unenforced, unmeasured, free**.

Include verbatim: **"An override is a signal about the skill, not misbehaviour by
the model."**

### 5.2 Override capture — no new plumbing

`runners/worker_init.py` `_build_prompt` gains one instruction for the line format:

```
SKILL-OVERRIDE: <skill> :: <rule> :: <did instead> :: <why>
```

`skill-report.py` reads those lines out of stores that **already exist** —
`state/tasks.db` `tasks.report` (582 populated rows) and `state/logs/cto.log`.
No new hook, no new log file, no settings paste, and it works retroactively.

### 5.3 Optional linter

`scripts/skill-doctrine-lint.py`, hand-run, listing HARD blocks missing a reason.
**One absolute defect count, threshold zero.** No `--strict`, no ratio.

**Not built:** `cage_ratio`. It had no variance (all 22 skills score identically
today), its 0.30 threshold was derived from nothing, its mandate matcher produced
unvalidated Thai false positives (`ต้อง` / `ห้าม`), and it had no consumer. Do not
resurrect it. There is also **no Tier-0 mechanical tagging commit** across the 21
skills — it existed only to de-noise that metric.

Retrofit is **on next edit**, not big-bang. One exception worth doing on its own:
fix the zero-digits vs strike-through contradiction inside
`.claude/skills/higgsfield-unlimited-gen/SKILL.md`, which is a correctness bug
regardless of this ADR.

---

## Suggested task split

| Task | Role | Wave | Depends on |
|---|---|---|---|
| Hook registration + role column + recovery | devops_engineer | 0 | — |
| Frontmatter contract + skill-lint + backfill | developer | 1 | 0 |
| Curator git verbs + sidecar deletion | developer | 2 | 1 |
| Visibility Phase 1 | devops_engineer | 3 | 0.3 |
| skill-report columns + objection tool | developer | 4 | 1, 2 |
| skill-author doctrine rewrite | prompt_engineer | 5 | 1 |

**Correction, 2026-09-01.** An earlier version of this table said Waves 1 and 3
Phase 1 both touch `runners/worker_init.py` and must be serialized. That was
wrong — Wave 1 never touches it (Wave 0 did, and Wave 0 merged first). The two
ran concurrently with disjoint locks and no collision. Verify a claimed overlap
against the actual declared `touches` before serializing; needlessly serialising
waves costs real wall-clock.

The role column above also originally read `devops_engineer` for Waves 0 and 3.
Both shipped as `developer`, because the deliverable was Python plus pytest, not
infrastructure. Role follows the deliverable, not the topic.
