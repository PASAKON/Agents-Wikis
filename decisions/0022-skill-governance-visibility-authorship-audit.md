# ADR 0022 — Skills get a per-role audience, open authorship, and git as their undo

- **Date:** 2026-09-01
- **Status:** Accepted (three items flagged for CEO sign-off, see §9)
- **Decider:** CEO, with CTO refinements accepted in session
- **Supersedes:** [ADR 0018](0018-skill-lifecycle-and-curator.md) §6, which put
  agent-authored skill creation out of scope. This ADR reverses that.
- **Source:** Hermes Agent re-survey 2026-09-01 (`reference/hermes-agent-vs-org.md`),
  plus 18 agents writing and adversarially critiquing six subsystem specs.
  30 blocker-level problems were raised against the first draft; every decision
  below is the post-critique version.

---

## 1. Context

### What the CEO decided

1. **Visibility is per-role.** Each role sees only the skills it needs.
2. **CTO is the exception** and sees all skills, because the CEO works through
   the CTO and the CTO must be able to audit everything.
3. **CTO grants** a skill to a role, at CTO + CEO discretion.
4. **Authorship is open.** Every role may write its own skill with **no
   pre-approval**. An approval gate was rejected outright: it slows the org down.
5. **Every skill is stamped** with who wrote it.
6. **The name carries a prefix** naming its audience
   (`CTO_Higgsfield_Seedance2.5_Prompt`). CTO proposed and CEO agreed that the
   prefix means **who uses it**, while **who wrote it** lives in a field.
7. **A C-level may write a skill for a worker.**
8. **A worker may object** to a skill it did not write when using it causes a
   conflict, so author and user improve it together.
9. **No approval system**, now or later, unless the CEO reverses this.

CTO proposals the CEO approved: an **undo** replaces the approval gate; skills
may **hide themselves**; the new-skill signal folds into the **audit report**;
and skills must not **cage the model** (§7).

### What we measured on 2026-09-01

| Fact | Value |
|---|---|
| Skills visible to a session | 293 |
| Never invoked, even once | 267 |
| Distinct skills ever fired / total fires | 48 / 422 |
| Skills we actually own | 22 (`Agents/.claude/skills`); the rest are symlinks into repos we do not own |
| Skills carrying `created_by` | **1 of 22** |
| `state/skill-usage.json` (the curator's sidecar) | **never written** — no lifecycle action has ever run |
| Curator verbs | `status`, `propose`, `archive`, `restore`, `pin`, `unpin`. No `create`, no `patch` |
| Skill mutations already in git history | **85, over two months, with zero maintenance** |

### Three live defects found while specifying this

- **Worker skill telemetry is being destroyed.** `scripts/hook-skill-log.py`
  resolves its log path relative to `__file__`, so from a worktree it writes to
  `worktrees/<...>/state/skill-usage.log`. **79 worker fires are stranded there
  right now** (browser-operator 38, higgsfield-unlimited-gen 36, gdrive-filing 5),
  and merge cleans the worktree. Cross-repo worktrees never run the hook at all.
- **`created_by` silently coerces.** `scripts/skill-curator.py:157-158` maps any
  value outside `{human, agent}` to `human`. A role-valued `created_by: cto`
  would quietly disable the curator on that skill.
- **`~/.claude/skills` holds symlinks into this repo** (`browser-operator`,
  `mooniex-finance`, ...). Renaming an org skill directory dangles the symlink
  and the skill **vanishes** with no error.

---

## 2. Decision — visibility is a settings file, not a symlink farm

**Mechanism, measured on Claude Code 2.1.252, not assumed.**

Claude Code merges a `skillOverrides` map across five settings sources in this
precedence: `policySettings` > **`flagSettings` (the `--settings` flag)** >
`userSettings` > `projectSettings` > `localSettings`. Values are
`on | name-only | user-invocable-only | off`.

| Configuration | Skills listed | Prompt tokens |
|---|---|---|
| Baseline, bare cwd | 170 | 44,147 |
| 42 `~/.claude/skills` names set `off` | 128 | 36,282 (−17.8%) |
| 4 plugins disabled via `enabledPlugins` | 57 | — |
| Both, a realistic worker profile | **18** | **27,124 (−38.6%)** |

Two levers, because plugin skills bypass the first: the effective gate
short-circuits with `if (e.type !== "prompt" || e.source === "plugin") return "on"`.
So **`skillOverrides` for normal skills, `enabledPlugins` for plugin skills**,
with keys `<plugin>@<marketplace>` read verbatim from
`~/.claude/plugins/installed_plugins.json`.

Because `flagSettings` outranks `localSettings`, a `--settings` file passed by
the spawner **outranks the `settings.local.json` a worker holds in its own
worktree**. A worker cannot lift its own cage.

**This corrects a standing belief.** Our記録 that `skillOverrides: "off"` does not
save context came from testing ECC skills, which are plugin skills and therefore
exempt. The mechanism was real; the generalisation was not.

**Shape, after the simplification critique:**

- Two or three **checked-in profile files** under `policies/skill-visibility/`
  (`worker-baseline.json`, `browser.json`, later `engineering.json`) — never one
  per role, never generated per worktree.
- **One key per role** in `policies/agents.yaml`:
  `skills_profile: worker-baseline | browser | all`.
- `all` **emits no file and passes no flag**, so CEO decision 2 (CTO sees
  everything) costs zero code.
- Rejected: `skill_bundles`, `external_map`, a `builtins:` enum, `requires_env`,
  a per-worktree generated file, and a `lib/skill_visibility.py` module.

**Phased, because the allow-lists must be earned:**

- **Phase 1 — plugins only.** 113 of 293 skills are plugin skills; disabling four
  plugins is most of the win for one list per role, with no enumeration problem.
- **Phase 2 — per-skill allow-lists**, only after `hook-skill-log.py` has carried
  a role column for two weeks, so each role's list is seeded from measured
  behaviour instead of a guess.

**Self-hiding (CEO proposal B)** rides the same file. A skill may declare
`requires_tools` / `requires_roles`; a skill whose requirements the role does not
satisfy is forced `off` even if granted. **Self-hiding may only subtract, never
add** — it is a safety narrowing, not a grant.

---

## 3. Decision — the field is the authority, the prefix is the mirror, and we do not rename

**Frontmatter contract** for an org-authored skill:

```yaml
name: <directory basename>
description: <=350 chars, the only body text that costs context
owner: CTO                      # existing; also the objection address
origin: mooniex-org             # existing
scope: >-                       # existing
audience: [browser_operator]    # NEW — CEO rules 1, 2, 7. Role keys, or all|cxo|worker
created_by: human               # NEW — EXACTLY human|agent. Anything else silently
                                #        coerces to human at skill-curator.py:157
author: {role: cto, date: 2026-09-01}   # NEW — CEO rule 5, the rich stamp
improved_by: []                 # NEW, optional, append-only — CEO rule 8
aka: []                         # NEW, optional — preserves telemetry across a rename
```

`created_by` stays a two-value enum because the curator's gate depends on it.
The **role** goes in `author`, never in `created_by`.

**Prefix: new skills only.** Renaming the 21 existing directories would dangle
the `~/.claude/skills` symlinks (the skill disappears silently), orphan 284
logged fires, and break 39 files that name skills as literal strings — including
prompt text inside `runners/secretary_server.py`. For zero functional gain, since
`audience:` is what any machine reads.

So: **`skill-author` names every new skill with its audience prefix from the day
the lint lands; the existing 21 are grandfathered and get `audience:` only.**
Reports render `cto / merge-checklist` computed from the field, which delivers
what CEO rule 6 is for — seeing at a glance who a skill is for — at no rename
cost. **This deviates from the letter of decision 6 and needs CEO sign-off (§9).**

**Enforcement is a lint, not a gate.** `scripts/skill-lint.py` with five codes
(missing SKILL.md, unparseable frontmatter, name ≠ directory, `created_by` outside
the enum, `audience` token unknown), wired into the **existing** `.git/hooks/pre-commit`.
Skills are committed files, so that is complete coverage with no settings paste.

---

## 4. Decision — git is the ledger. We are not building one.

CEO decision A asked for an append-only ledger plus before/after copies so any
change can be rolled back. The first spec proposed `state/skill-ledger/*.jsonl`
plus a content-addressed blob store. The critique returned **wrong-approach**,
and it is right:

- `.claude/skills/` is **already git-tracked**.
- Commits are append-only; `merge_task` already uses `--no-ff` (`git_ops.py:80,212`)
  so branch commits survive.
- Git objects **are** the before/after copies, compressed and deduplicated.
- `git revert` is the rollback; `git log --follow -- .claude/skills/<name>` is the
  history verb; `git status --porcelain -- .claude/skills` is drift detection.
- **85 skill mutations are already stored this way**, over two months, with zero
  maintenance verbs, zero caps and zero garbage collection.
- `scripts/skill-curator.py:409` **already** does a full `copytree` backup before
  every mutation. A blob store would have been the **third** redundant copy.

**Decision:** every mutating curator verb ends by committing the skill directory
with a `Skill-Actor: <role>/<session>` trailer, and we add three thin read verbs
(`history`, `undo`, `drift`). Roughly 55 lines instead of a subsystem.

Deleted before being written: `lib/skill_ledger.py`, `state/skill-ledger/`,
`state/skill-blobs/`, sharding, blob caps, `gc-blobs`, retention tuning, and the
`_touches_violation` allowlist. That removed 7 of 8 listed risks by not creating
them.

**Two honest limits**, neither of which the bespoke ledger solved better:

- A skill edited and never committed is unrecorded. CEO rule 9 forbids the gate
  that would force it; `drift` catches it as one `git status` call.
- Lifecycle state moves into each SKILL.md's frontmatter. `state/skill-usage.json`
  and its `_load_state`/`_save_state` are **deleted** — the file was never written,
  so there is nothing to migrate.

**One consequence:** org skills must live **in the repo**. `skill-author`'s
`~/.claude/skills/<name>/` location is outside git, outside the curator's reach
and outside this ledger by construction. That path is removed for org-authored
skills.

---

## 5. Decision — fix the hook first; the audit is three columns after that

Most of the proposed audit machinery existed to route around one bug.

**Fix, once:** register the `PostToolUse` Skill hook in **`~/.claude/settings.json`**
with an **absolute** script path and an absolute default log path. That single
change removes the need for git-common-dir discovery, an `ORG_SKILL_LOG` export,
a reclaim cadence, and it additionally covers the 16 cross-repo worktrees that
have no `.claude/settings.json` of their own.

**Then:**

- The log grows from 3 fields to 4: `ts \t skill \t session \t role`. Legacy
  3-field lines pad. Without the role column, no allow-list in §2 Phase 2 can
  ever be evidence-based.
- One shell line, run once, recovers the 79 stranded worker fires.
- `scripts/skill-report.py` gains three columns: **`/wk`** (fires per week since
  that skill's own first fire, which is the "how often" the CEO asked for and
  which Hermes cannot compute at all), **`updated`**
  (`git log -1 --format=%cI -- SKILL.md`, which works through the external
  symlinks), and **`obj`** (open / total objections, §6).
- `--brief` prints **zero bytes when there is nothing to say**, and that is the
  one line `/session-open` shows. This is CEO proposal C, folded in rather than
  built separately.

**Guard against the obvious wrong move.** Hermes' own curator says: *"'use=0' is
not evidence a skill is valuable; it's absence of evidence either way."* We adopt
that as a rule: silence alone never archives a skill.

**On "how often it goes wrong."** Nobody has this, Hermes included — every one of
their counters sits inside a success gate, so a failed call writes nothing and
they measure *whether a skill was opened*, never *whether it worked*. We build the
honest version: **objections are the signal** (§6), and **task outcome is smoke**
(a task that fired skill X and ended `failed`/`stalled`). The join must be
verified before it is relied on; smoke is reported as smoke, never as a defect rate.

---

## 6. Decision — objections are one events row and one to-do

No new transport. No four-rung author-liveness ladder, no tmux delivery, no
mailbox — measured, most objections route to the CTO regardless of which rung
fires, and ADR 0017 already accepts that a human runs the verb.

`tools/skill_objection.py`, roughly 80 lines:

- `raise_objection(skill, conflict, did_instead, blocked)` resolves the author
  from frontmatter (absent → CTO), writes **one `skill_objection` event** via the
  existing `db.log_event` (no DDL), and adds a **LungNote to-do** addressed to
  that role.
- `summary(days)` feeds the `obj` column in §5.
- Resolution is **completing the to-do**. There is no separate resolve verb.
- One MCP tool so a worker can call it, granted in the same commit.

**The load-bearing part is not the code.** `skill-author` must require an
**"If this skill is wrong"** section in every skill body. A channel nobody is
told about gets zero traffic, and it costs one paragraph.

Author backfill is **lazy**: a skill gains `author:` the next time someone edits
it. Until then it routes to CTO, which is where it was going anyway.

---

## 7. Decision — a skill must not cage the model

**The principle, in the CEO's own framing:** a skill that says "you must do
exactly X" removes the model's judgement. That costs nothing on a weak model and
costs real capability on a strong one. **We run only frontier models.**

This is not something we can copy. Hermes does the **opposite, deliberately**:
its system prompt tells the model to load a skill *"even if you think you could
handle the task with basic tools"* and *"even for tasks you already know how to
do"*; across 58 bundled skills there are roughly 460 mandate tokens against 4
uses of the word "judgment" and none of "discretion"; the only relief valves are
ask-the-human or patch-the-skill, never deviate. That posture is the **price of
supporting 300+ models from a local Llama to Opus** — they must write for the
weakest model they support. We do not pay that price, so we should not accept its
side effect. Their own maintainer proves it is a choice: the songwriting skill
opens *"Everything here is a GUIDELINE, not a rule."*

**The contract, binary after critique:**

> A rule block is either **HARD**, and then it carries a **`Why hard:`** clause,
> or it is **advice the model may override** — and when it overrides, it says why.

Money, irreversible actions and safety stay HARD.

FACT (things the model cannot derive) and WHY (cause → effect it can reason from)
stay in `skill-author` as **authoring advice — unenforced, unmeasured, free**.
Nothing in the mechanism ever read them differently, so tiering every directive in
the corpus would have bought nothing.

**`cage_ratio` is not built.** All 22 org skills score identically today, the
0.30 threshold was not derived from anything, the mandate matcher produced Thai
false positives (`ต้อง` / `ห้าม`) that were never validated, and it had no
consumer. The linter instead reports **one absolute defect count, threshold zero:
HARD blocks missing a reason.** A skill with no HARD blocks scores zero defects on
day one, which also removes the need for a 21-file mechanical tagging commit.

**The lint is hand-run and advisory, never a gate.** A `--strict` lint inside the
authoring flow is an approval gate wearing a different hat, and CEO decisions 4
and 10 forbid it.

**Overrides are captured from stores that already exist.** A `SKILL-OVERRIDE:`
line in the worker's report lands in `tasks.report` (582 populated rows today)
and `state/logs/cto.log`; `skill-report.py` reads them. No new hook, no new log
file, no settings paste — and it works **retroactively**.

**An override is a signal about the skill, not misbehaviour by the model.** That
sentence goes in `skill-author` verbatim.

---

## 8. Consequences

- Worker sessions get roughly **38% of their prompt back** once Phase 2 lands,
  every session, forever. Per-role scoping is a cost fix, not only governance.
- The curator finally has input. It has archived nothing so far because 21 of 22
  skills carry no `created_by`, so the gate correctly refuses everything.
- Three subsystems that were specified are **not being built**: the ledger, the
  blob store, and `cage_ratio`. Git and existing stores cover them.
- Three live defects (§1) get fixed as **Wave 0**, before any new feature.
- We inherit Hermes' hardest limitation only where it is genuinely hard: a skill
  edited and never committed is unrecorded.

---

## 9. Needs CEO sign-off

1. **Prefix on new skills only.** Renaming the existing 21 dangles symlinks,
   orphans 284 fires and touches 39 files, for no machine-readable gain. Proposal:
   prefix every new skill immediately, grandfather the existing 21 permanently,
   revisit after 90 days of measured `audience` data.
2. **Phase 2 waits two weeks.** Per-skill allow-lists are seeded from a measured
   role column rather than a guess. Phase 1 (plugins) ships immediately and is
   most of the context win.
3. **`skill-author` loses its global write path.** Org skills must land in the
   repo, or they sit outside git, outside the curator and outside undo.

---

## 10. Explicitly not doing

- No approval gate, in any form, including a `--strict` lint in the authoring flow.
- No daemon and no autonomous curation ([ADR 0017](0017-no-autonomous-task-dispatch.md) stands).
- No bespoke ledger or blob store.
- No `cage_ratio` metric.
- No bulk rename of existing skills.
- No copying Hermes' "load the skill even if you could do it yourself" posture.
