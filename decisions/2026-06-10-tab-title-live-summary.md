# ADR 2026-06-10 — C-level tab title = live status + session summary

**Status:** accepted · **Author:** CTO · **Driver:** CEO directive 2026-06-10

## Context

C-level iTerm tabs were titled once at spawn (`CTO Chat #<id>`) and never
updated. With several C-level + DEV tabs open the CEO cannot tell which
session is doing what, which needs attention, and which can be closed.

Tab-title matching is load-bearing in the org plumbing:

- `tools/send_to_cto.py` routes DEV reports into the CTO tab by title substring
- `tools/delegate.py` clusters DEV tabs under the owning CTO's window
- `tools/send_to_cxo.py` types CEO/CTO messages into the primary CXO tab
- `tools/itermtab.py:close_tab` closes DEV tabs whose title contains a task-id

Any title format change therefore risks orphaned DEV reports.

## Decision

1. **Title format** `<ROLE> #<session_id> <glyph> <summary>`. Base prefix
   stays stable for routing; glyph + summary update at every finished job
   (IRON-RULES §32). Five glyphs: ⏳ working · ✅ batch done, more pending ·
   🔴 blocked · 💤 idle · 🏁 **all assigned work done, zero blockers —
   session safe to close** (CEO-requested closing signal).
2. **Helper** `Agents/scripts/tab-title.sh` — writes OSC-0 directly to the
   tty saved at launch (`state/locks/<role>-<sid>.tty`); AppleScript fallback
   via saved winid. Persists `state/tab-titles/<role>-<sid>.{base,title}`.
   A 60s keeper loop in `cto-claude.sh` / `cxo-claude.sh` re-asserts the
   saved title (survives zsh precmd + claude CLI title rewrites); launchers
   also set `CLAUDE_CODE_DISABLE_TERMINAL_TITLE=1`.
3. **Matchers extended, not replaced** — `send_to_cto` / `send_to_cxo` /
   `delegate` match legacy `<X> Chat #<id>` OR new `<X> #<id>`, so in-flight
   DEVs and pre-rollout tabs keep routing during transition. `close_tab`
   gains a C-level guard (never closes a tab titled `CTO/CMO/CGO/CFO …`)
   because summaries are free text.
4. **Doctrine** appended to `roles/cto|cmo|cgo|cfo.md`; summaries must never
   contain task-ids; tests in `Agents/scripts/test_tab_title.py`.

## Consequences

- CEO scans the tab bar → instant per-session status; 🏁 tabs close safely.
- Old-format tabs keep working; no orphaned DEV reports during rollout.
- Glyph discipline is agent-side (role doctrine); the keeper loop only
  re-asserts, it cannot invent status.
- `*`/✳ busy indicator from claude CLI is superseded by our glyphs when
  `CLAUDE_CODE_DISABLE_TERMINAL_TITLE` is honored.
