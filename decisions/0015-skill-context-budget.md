# ADR 0015 — Skills are a per-session context budget, not a free shelf

- **Date:** 2026-08-07
- **Author:** CTO (session #4bf8b5a9), at CEO request
- **Status:** Accepted
- **Companion to:** [0014 CXO strict MCP loading](0014-cxo-strict-mcp-loading.md) —
  same failure mode (unbounded install, per-session cost), different surface.
- **Implements:** `skills-registry.md` (the operational rules this ADR justifies)

## Context

The CEO could not tell where any given skill came from, what it was allowed to
do, or why so many had near-identical names. Measuring the install rather than
guessing:

| Always-loaded surface | Count | Chars |
|---|---:|---:|
| Wealth-management skills symlinked into `mooniex-agents/.claude/skills/` | 92 | 65,455 |
| ECC plugin skills | 229 | 62,947 |
| User-scope skills (`~/.claude/skills`) | 58 | 9,638 |
| Org's own skills | 13 | 8,642 |
| `~/.claude/CLAUDE.md` | — | 10,939 |
| **Total** | **~450** | **~158,000 (~45k tokens)** |

Roughly 45k tokens were consumed before the first user message of every session.

Three findings drove the decision, each measured rather than assumed:

1. **Only `name` and `description` are injected.** ECC declares `origin:` in 453
   of its SKILL.md files and it never appears in the session listing. Provenance
   metadata and skill bodies are therefore free; `description` is the sole cost.
2. **`skillOverrides: "off"` does not reduce context.** 76 ECC skills were
   already set to `off` in `~/.claude/settings.json`, and every one still
   appeared in the listing. It suppresses auto-invocation only. Those 76 entries
   were paying full price for zero benefit.
3. **Install has no relationship to use.** Against `skillUsage` telemetry in
   `~/.claude.json` across all recorded history: of 229 ECC skills, **3 had ever
   been invoked**, once each, months prior. The one heavily-used ECC entry,
   `save-session` (40 uses), is a **command**, not a skill — and so was never
   part of the cost at all.

A fourth finding explains the naming mess: the `mooniex-` prefix and the
keyword-stuffed descriptions existed to win disambiguation fights against
`ecc:seo`, `paid-advertising`, `analytics` and friends. The routing table in
`~/.claude/CLAUDE.md` — 90 of its 173 lines — existed for the same reason. All
of it was compensation for an over-broad install, not a naming problem.

## Decision

1. **Treat `description` as a budget.** Target ≤ 350 chars, English prose, Thai
   trigger phrases kept verbatim (they are matcher data, not prose). Push detail
   into the body, which is free.
2. **Every org-owned skill declares `owner`, `origin`, and `scope`.** `scope`
   states what the skill can and cannot do. Free at load time, and it closes the
   gap that made skills unreadable from outside — `mooniex-image-gen` can reach
   MeiGen's generate tool but no API key exists, so only the free search/enhance
   half is usable and the prompt goes to Fal.ai. Nothing had said so.
3. **Disambiguation lives in the skill, not in CLAUDE.md.** Each skill's
   `description` carries its own "Use instead of X" clause. The per-domain
   routing table was deleted; what remains covers only skills we do not own and
   therefore cannot annotate.
4. **CLAUDE.md holds only what is true in every session regardless of task.**
   Everything else is a skill, which loads on demand.
5. **Uninstall, don't disable.** Since `off` does not help, unused skills are
   moved out of the skills directory entirely — parked, never deleted, with a
   `RESTORE.md` beside them.
6. **Public/private is judged per skill on content, not on name.** See
   `skills-registry.md`. `mooniex-finance` stayed private (card last-four digits
   mapped to vendors, a receipt number, per-brand budget caps); a public push
   cannot be taken back.
7. **Naming follows recall, not conflict-avoidance** — possible only once the
   install shrank enough that no literal collisions remained. Prefix generic
   domain words, drop the meaningless `-skill` suffix, no prefix on names that
   are already unambiguous.

## Consequences

| | Before | After |
|---|---:|---:|
| Skills in the listing | ~450 | ~84 |
| Description chars | 147,000 | 29,658 |
| `~/.claude/CLAUDE.md` | 10,939 | 4,113 |
| **Combined** | **157,939** | **33,702** |

A **79% cut, roughly 31,000 tokens returned to every session.**

- ECC keeps its `agents/` (60), `commands/` (75), `hooks/` (3) and `.mcp.json`.
  Only `skills/` was trimmed, 229 → 19. **The GateGuard `gateguard-fact-force`
  hook the CEO relies on is an ECC hook** — disabling the plugin outright would
  have killed it. That is why the plugin stays enabled.
- Parked, reversible, with restore instructions:
  `~/.claude/ecc-skills-parked-2026-08-07/` (210) and
  `~/.claude/superseded-skills-parked-2026-08-07/` (12).
- Both park directories live in a plugin **cache** path or under `~/.claude`. An
  ECC version bump installs to a new `cache/ecc/ecc/<version>/`, so the trim does
  not carry forward — re-apply after upgrading. Same caveat as ADR 0014.

## What this does not fix

The remaining cost is concentrated in user scope (`~/.claude/skills`), which is
global and touches the CEO's personal sessions, so it was deliberately left
alone beyond removing entries already documented as superseded. Several skills
there still have no recorded origin.

## Verification

Live, mid-session: the harness re-injected the skill listing three times during
the work and each time reflected the change — the 91 wealth-management entries
disappeared, the rewritten descriptions appeared, and a skill with malformed YAML
vanished from the listing until fixed. All 21 org skills parse as valid YAML with
`name` matching their directory and their `Trigger on /…` clause intact; 12
spot-checked Thai trigger phrases survived; skill bodies are byte-identical.

## Lessons worth keeping

- **Measure the install against usage telemetry before pruning.** `skillUsage` in
  `~/.claude.json` turned a judgement call into arithmetic.
- **A config flag that claims to disable something may only disable part of it.**
  `skillOverrides: "off"` was trusted for 76 skills and delivered nothing.
- **A cost that is invisible per-item is still real in aggregate.** No single
  skill looked expensive; 450 of them cost a quarter of a context window.
- **Documented intent beats inference when pruning.** One skill
  (`content-idea-generator`) was parked on my own reading of "looks superseded"
  rather than on the registry, which recorded it as a proprietary purchase kept
  deliberately. Caught on review and restored. The other 15 were each backed by
  an explicit prior note.
