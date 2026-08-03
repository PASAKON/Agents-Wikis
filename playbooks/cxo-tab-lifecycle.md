---
title: CXO tab lifecycle — spawn, dedupe, ping, close
tags: [playbook, cto, cxo, iterm, lifecycle]
status: Active
date: 2026-05-26
owner: CTO
---

# CXO tab lifecycle

> **Problem this fixes (CEO 2026-05-26):**
> When CTO sends work to CMO/CFO via `tools/send_to_cxo.py`, the message
> currently types into the C-level's primary iTerm tab. If CEO is mid-
> conversation with CFO in that same tab, CTO's request collides — chat
> threads, contexts, and task threads get mixed.
>
> Solution: every CTO→CXO request opens a **new ephemeral tab**, like
> DEV. Each tab is owned by exactly one session id. Auto-close is
> ID-locked: a CXO can only close its own tab, never a sibling's or
> a DEV's. Spawn is dedupe-checked first.

## Identity model

Every C-level iTerm tab is keyed by `<role>-<session_id>`:

- `state/locks/<role>-<session>.winid` — file with iTerm window id
- `state/locks/<role>-active` — pointer to CEO's primary session for that role
- `$CXO_ROLE` + `$CXO_SESSION_ID` — env baked into each tab on spawn

These four facts together prove ownership. **No tab is closed without
all four matching.**

## Spawn protocol (pre-check before opening new tab)

When CTO needs to message CMO/CFO/CGO:

1. Read `state/locks/<role>-*.winid` for every lock file of that role.
2. For each, run `osascript -e 'tell application "iTerm2" to return id of windows'`.
3. Classify each lock as:
   - **Alive + topic-match** → reuse this tab (send message normally)
   - **Alive + topic-mismatch + idle** → ping the tab (see "Idle ping" below)
   - **Alive + topic-mismatch + busy** → spawn a NEW ephemeral tab
   - **Dead (winid not in live list)** → garbage-collect the lock file, no close needed
4. If spawning new: use `scripts/cxo-claude.sh <role> --session <ephemeral_id> --initial-prompt "<msg>"` so the message arrives with the tab.

**Tab title format** for ephemeral tabs:
- `CFO <- CTO: <topic-slug>` (`<topic-slug>` = first 30 chars of message, kebab-cased)
- Makes ownership + provenance visible to CEO at a glance.

**Primary tab (`<role>-active`) is sacrosanct.** Never overwrite that
pointer with an ephemeral session id. CEO's chat thread stays clean.

## Idle ping (before any auto-close)

Before closing a tab on idle or timeout, the owner sends a ping into
its own tab:

```
[from <SENDER-ROLE>]: ยังทำงานต่ออยู่ไหม? ไม่ตอบใน 5 นาที = close session.
```

- Reply "ยัง" / "yes" / "still working" -> cancel close, reset 5-min timer
- No reply within 5 min -> close session + tab
- Reply "เสร็จแล้ว" / "done" -> close immediately

This prevents the CTO from killing a DEV mid-thought just because the
DEV is reading code instead of typing.

## Close protocol (ID-locked, own-tab-only)

A session may only close a tab when **all** these are true:

1. `state/locks/<role>-<session>.winid` exists and contains a winid.
2. That winid is currently in iTerm's live window list.
3. The caller's `$CXO_ROLE` matches `<role>` AND `$CXO_SESSION_ID` matches `<session>`.
4. The session has no in-flight task with `status=in_progress` in the org DB.

If any fails -> refuse to close. Log the refusal to `state/locks/close-refusals.log`.

**Helper:** `tools/itermtab.py::close_session(session_id, owner_role)`
already enforces (1)-(3); the in-progress check (4) is the new addition.

## Garbage collection (stale locks)

A `*.winid` is "stale" when:
- The winid is NOT in iTerm's current `id of windows` list.

Stale locks are safe to delete — they reference a window that no longer
exists. The cleanup script `scripts/cleanup-zombies.sh` does this scan
and removes stale lock files only. It **never** closes a live tab; it
only deletes file references.

Run cleanup:
- On CTO session start (boot hook)
- On demand: `bash scripts/cleanup-zombies.sh`

## Safety rules (HARD)

1. **ID-match-only close.** Never close a winid that doesn't appear in
   the caller's own lock file.
2. **Never close a tab with `status=in_progress` task.** Even if idle.
3. **Never close the `<role>-active` primary tab from a sibling C-level.**
   Only CEO or that role's own session may close its primary.
4. **Ping before timeout-close.** Default 5-min wait for reply.
5. **DEV tabs follow the same rules** — owned by their task_id, only
   the DEV's own claude process (or CTO via `merge_task`) may close.

## See also

- [[cto-dev-orchestration]] — DEV tab spawn (parallel pattern)
- [[agent-messaging]] — coord MCP messaging (async queue alternative)
- [IRON-RULES §29 DEV visible animation](https://github.com/PASAKON/MoonieX-Wikis/blob/main/IRON-RULES.md#section-29) (`mooniex:IRON-RULES.md`)
- [[../decisions/2026-05-26-cxo-tab-id-locked-close|ADR 2026-05-26]]
