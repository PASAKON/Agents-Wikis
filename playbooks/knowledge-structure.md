---
title: Knowledge structure — per-role banks, open-design pattern
tags: [playbook, knowledge, workers, c-level, claudesign]
status: Active
date: 2026-05-26
owner: CTO
---

# Knowledge structure

> **Goal:** every role that produces brand-strict, formula-strict, or
> playbook-driven output has a **dedicated knowledge bank** modeled
> after the open-design pattern in
> [`mooniex-claudesign`](https://github.com/PASAKON/mooniex-claudesign).
> Workers read it via symlink injection on `delegate_task`; C-level
> reads it directly.

## Why this exists

`web_designer` agents already get a knowledge bank
(`/Users/gob/Projects/mooniex-claudesign/`) symlinked into their
worktree as `.claudesign-knowledge/`. They use it to pick the right
artifact-shape skill, brand design system, and craft rules — and they
ship better-aligned work because the bank is the source of truth.

Other roles (`ads_manager`, `content_strategist`, `data_analyst`,
plus the CFO surface) have no equivalent today. They reinvent
patterns, drift from brand voice, miss craft rules, and have no place
to record reusable playbooks.

This playbook documents the shape so every new bank looks the same and
every agent reads it the same way.

## Location

Two valid locations:

| Pattern | Location | Used by |
|---|---|---|
| **External repo** (legacy) | `/Users/gob/Projects/mooniex-<bank>/` | `web_designer` (`mooniex-claudesign`). DO NOT migrate this path — keep as-is. |
| **In-tree subfolder** (canonical going forward) | `/Users/gob/Projects/Agents/knowledge/<role>-knowledge/` | All new banks. |

Both patterns coexist. `KNOWLEDGE_MAP` in `runners/dev_init.py` points
each role at the right path so symlink injection works regardless.

## Anatomy (mirror claudesign)

Every bank follows the same three-layer shape:

```
<role>-knowledge/
  ├─ skills/                # artifact-shape playbooks
  │  └─ <skill-name>/
  │     ├─ SKILL.md         # frontmatter + body + acceptance
  │     └─ example.*        # reference output (html / json / md / sql / ipynb)
  ├─ <brand-tier-dir>/      # role-specific dimension (one folder per brand or tier)
  ├─ craft/                 # universal anti-patterns + invariants
  │  └─ *.md
  ├─ AGENTS.md              # how agents should use this repo
  └─ CLAUDE.md              # rules + read-order for this role
```

### The three layers

1. **`skills/`** = **WHAT to make.** One folder per artifact shape. Each
   folder has `SKILL.md` with frontmatter (`name`, `description`,
   `triggers`, `acceptance`) + an `example.*` reference. Skills are
   composable but not stackable: one task = one primary skill.
2. **`<brand-tier-dir>/`** = **HOW it should look / sound / measure.** The
   role-specific brand layer:
   - `ads_manager` → `campaigns/<brand>/CAMPAIGN.md`
   - `content_strategist` → `voices/<brand>/VOICE.md`
   - `data_analyst` → `models/<brand>/MODEL.md`
   - `finance` → `envelopes/<brand>-FY<year>.md`
   - `web_designer` (claudesign) → `design-systems/<brand>/DESIGN.md`
3. **`craft/`** = **what NOT to do.** Universal anti-patterns + linted
   rules. Skim every file every task. claudesign's `craft/anti-ai-slop.md`
   is the gold standard — 7 cardinal sins, each auto-checkable.

### `CLAUDE.md` (per bank)

The bank's own read-this-first. Contents:
1. Read order (workers MUST follow)
2. How C-level uses the bank (direct read, not symlink)
3. Hard rules (auto-reject criteria)
4. See-also links to role md + wiki

### `AGENTS.md` (optional per bank)

Same target audience as CLAUDE.md but framed for agent-to-agent
collaboration: which task types map to which skill, when to escalate,
who owns updates. Skip for banks where CLAUDE.md already covers it.

## Access matrix

| Reader | Path used | Mechanism |
|---|---|---|
| **Worker** in worktree | `worktree/.<bank>-knowledge/` (symlink) | `runners/dev_init.py::KNOWLEDGE_MAP` injects on spawn |
| **CTO** (this session) | `/Users/gob/Projects/Agents/knowledge/<role>-knowledge/` (direct) | Standard `Read` / `Bash cat` |
| **CMO / CGO / CFO** (own session) | same direct path | Standard `Read` / `wiki_read` not needed (knowledge is repo, not wiki) |

The C-level path is the **truth**. The symlink in worker worktree points
at the same files; workers can read but cannot write (workers are
`workspace_only` per `policies/agents.yaml`).

## Adding a new bank

1. **Create subdir:** `mkdir -p knowledge/<role>-knowledge/{skills,craft,<brand-tier>}`
2. **Write `CLAUDE.md`** with read order + hard rules + see-also.
3. **(Optional) Write `AGENTS.md`** if collaboration semantics matter.
4. **Seed `craft/anti-<role>-slop.md`** with the role-specific anti-patterns
   (mirror claudesign's `anti-ai-slop.md` style).
5. **Add 2-3 initial skills** with `SKILL.md` + `example.*`.
6. **Update `runners/dev_init.py::KNOWLEDGE_MAP`:**
   ```python
   KNOWLEDGE_MAP = {
       "web_designer": {
           "source": "/Users/gob/Projects/mooniex-claudesign",
           "mount":  ".claudesign-knowledge",
           "subdirs": ["skills", "design-systems", "craft"],
       },
       "ads_manager": {
           "source": "/Users/gob/Projects/Agents/knowledge/ads-knowledge",
           "mount":  ".ads-knowledge",
           "subdirs": ["skills", "campaigns", "craft"],
       },
       # ...add new role here
   }
   ```
7. **Update `roles/<role>.md`** Pre-work Checklist to include:
   ```
   N. Read `.<bank>-knowledge/CLAUDE.md` first, then the matching skill
      and brand-tier doc for your task.
   ```
8. **Smoke test:** spawn a `<role>` task and confirm the symlink
   resolves + worker reads from it.

## Adding a skill (any bank)

```
<bank>/skills/<skill-name>/
  ├─ SKILL.md     # frontmatter:
  │               # ---
  │               # name: <skill-name>
  │               # description: <one sentence>
  │               # triggers: ["keyword", "phrase"]
  │               # acceptance:
  │               #   - <criterion>
  │               # ---
  │               # # Body
  └─ example.*    # reference output, same format as the artifact
```

## Adding a craft file

Each `craft/*.md` is a **must-follow** doc. Workers read all of them
every task. Rules:

- One topic per file. Don't bundle "color" + "typography" in one doc.
- Each rule should be checkable (one-line manual or linter).
- Reference a real incident or principle. Theory-only rules drift.
- Keep <300 lines per file.

## Adding a brand-tier doc

| Role | File template |
|---|---|
| ads_manager | `campaigns/<brand>/CAMPAIGN.md` — objective, audience, KPIs, channel mix, budget envelope, creative-bank link |
| content_strategist | `voices/<brand>/VOICE.md` — tone, banned words, sentence rhythm, signature moves, hook bank |
| data_analyst | `models/<brand>/MODEL.md` — KPI formulas, attribution window, retention shape, segmentation rules |
| finance | `envelopes/<brand>-FY<year>.md` — total budget, per-category cap, approval threshold |

Owner: the C-level whose domain it lives in (CMO for campaigns/voices,
CGO/CFO for models, CFO for envelopes).

## Anti-patterns

- **Don't put strategy in `craft/`.** Craft is universal rules. Strategy
  (campaign approach, voice tone) lives in the brand-tier dir.
- **Don't duplicate wiki ADRs.** ADRs live in `LLMs/decisions/`. Knowledge
  banks reference them but don't re-encode them.
- **Don't bundle 10 skills in one folder.** One skill = one folder. The
  proliferation is the point — discoverable + greppable.
- **Don't gate-keep with permissions.** Every role's knowledge bank is
  readable by every C-level. Curation is by owner, not access.

## Source-of-truth precedence (when wiki + bank disagree)

| Surface | Wins on |
|---|---|
| Knowledge bank (`<role>-knowledge/`) | **Execution detail** (how to do the work) |
| Wiki ADR (`LLMs/decisions/`) | **Strategic decision** (why we made the choice) |
| Code (live in repo) | **Brand truth** (final hex tokens, real components) |

When in doubt: **code > bank > wiki**. Update the higher-precedence
surface first, then propagate down.

## See also

- [`mooniex-claudesign/CLAUDE.md`](https://github.com/PASAKON/MoonieX-Wikis/blob/main///Users/gob/Projects/mooniex-claudesign/CLAUDE.md) (`mooniex://Users/gob/Projects/mooniex-claudesign/CLAUDE.md`) — reference implementation
- [`knowledge/README.md`](https://github.com/PASAKON/MoonieX-Wikis/blob/main///Users/gob/Projects/Agents/knowledge/README.md) (`mooniex://Users/gob/Projects/Agents/knowledge/README.md`) — in-tree index
- [[cto-dev-orchestration]] — DEV spawn pattern (where symlink injection happens)
- [[cxo-tab-lifecycle]] — ephemeral tab pattern (when CTO consults CMO for brief auth)
