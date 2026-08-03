---
title: "ADR 2026-05-26 — CXO tab lifecycle: ID-locked spawn, ping, close"
tags: [adr, cto, cxo, iterm, lifecycle]
status: Accepted
date: 2026-05-26
---

# ADR — CXO tab lifecycle: ID-locked spawn, ping, close

## Status

Accepted

## Context

`tools/send_to_cxo.py` currently types CTO->CMO/CFO messages into the
recipient's **primary** iTerm tab (the one pointed at by
`state/locks/<role>-active`). CEO uses that same tab for direct chat
with the C-level. Result: when CTO sends a task while CEO is talking
to CFO, the two threads mix in one terminal window.

Symptoms CEO reported 2026-05-26:
- "งานมันปนกันมั่วไปหมด" — CFO conversation interleaved with CTO requests
- Stale `*.winid` lock files accumulate across sessions (6 found in
  `state/locks/`, 1 stale — winid 2354 for `cto-eed05c9d` referenced a
  window no longer in iTerm)
- No idle-ping protocol — CTO/CXO might close a tab while a DEV is
  reading code mid-task

## Decision

Adopt **ID-locked tab lifecycle** for every C-level and DEV tab:

1. **Spawn is dedupe-checked.** Before opening a new tab, the caller
   inspects existing `<role>-*.winid` files, classifies them against
   iTerm's live window list, and either reuses, pings, spawns new, or
   garbage-collects the lock file.

2. **CTO->CXO defaults to ephemeral spawn**, not type-into-primary.
   Each request lands in a new tab titled `CFO <- CTO: <topic-slug>`.
   CEO's primary CFO/CMO/CGO tab is never overwritten by sibling traffic.

3. **Close is ID-locked.** A session may close a tab only when:
   - The `<role>-<session>.winid` lock file exists and the winid is live
   - The caller's `$CXO_ROLE` + `$CXO_SESSION_ID` env match the lock owner
   - No `status=in_progress` task is bound to that session

4. **Idle ping before timeout-close.** Default 5 minutes wait for reply
   ("ยัง"/"เสร็จแล้ว"). No reply -> close. Prevents killing DEVs that
   are reading rather than typing.

5. **Stale-lock GC is file-only.** `scripts/cleanup-zombies.sh` removes
   `*.winid` files whose winid is not in iTerm's live list. It does NOT
   close windows.

Full operational detail lives in [[../playbooks/cxo-tab-lifecycle]].

## Consequences

**Positive**
- CEO's primary CFO/CMO chat tab stays single-threaded (one human + one
  C-level, no CTO interjection)
- Each CTO->CXO request is its own tab, traceable by topic-slug in title
- No more accidental cross-role tab closure (DEV killed by CTO, CFO
  killed by CGO, etc.) because close requires `$CXO_*` env match
- Stale `*.winid` files no longer accumulate across sessions

**Negative**
- More iTerm tabs open at peak (each CTO->CXO request = +1 tab during
  worker lifetime). Mitigated by auto-close on done + 5-min idle GC.
- Slightly more wall-clock latency per CTO->CXO message (claude spawn
  overhead ~2-3s) vs. typing into existing tab (~0s)
- Implementation touches `tools/send_to_cxo.py` + `scripts/cxo-claude.sh`
  + `tools/itermtab.py` + new `scripts/cleanup-zombies.sh`. Net ~80-150
  LOC across 4 files; one CTO patch task.

## Rollback Plan

If ephemeral-spawn pattern causes more churn than it solves:

1. Add `--no-spawn` flag to `send_to_cxo.py` defaulting to ON (revert
   to type-into-primary behavior)
2. Keep the close-protocol safety rules — they are net-positive even
   without ephemeral spawn
3. Document rollback in supersede ADR

## Alternatives Considered

- **Keep current type-into-primary, add "[CTO request]" prefix only** —
  rejected: still mixes threads in one terminal, CEO still has to mentally
  demux. Doesn't fix the underlying single-tab-per-role bottleneck.
- **Route everything through `mooniex-coord` MCP queue, no iTerm tab** —
  rejected for now: coord MCP `/api/office/directory` deployment is
  pending (CFO carry-over issue #8). When it ships, may revisit.
- **One persistent worker tab per role, message queue inside the tab** —
  rejected: complicates close semantics, recipient must demux topics
  inside their own conversation context, defeats the "clean chat" goal.

## Open Questions

- **Topic-slug collision:** if two CTO requests have identical first-30
  chars, tabs will share a title. Acceptable for MVP; revisit if it
  causes ambiguity.
- **`status=in_progress` check** requires CXO sessions to register their
  task on spawn the same way DEVs do. Schema migration may be needed in
  `state/agents.db` if `c_level_sessions` table doesn't already track
  active task id. Verify before implementation.

## See Also

- [[../playbooks/cxo-tab-lifecycle]] — operational detail
- [[../playbooks/cto-dev-orchestration]] — DEV spawn parallel pattern
- [[../playbooks/agent-messaging]] — async coord MCP alternative
- [IRON-RULES §29 DEV visible animation](https://github.com/PASAKON/Agents-Wikis/blob/main/IRON-RULES.md#section-29) (`org:IRON-RULES.md`)
- mooniex-agents tracker: see CFO carry-over issue #8 for coord MCP deployment status
