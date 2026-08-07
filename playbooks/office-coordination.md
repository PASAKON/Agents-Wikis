# Office coordination — RETIRED 2026-08-07

> **Status: unwired.** The four office hooks were removed from
> `~/.claude/settings.json` on 2026-08-07 (CEO decision). The scripts still
> exist on disk and nothing was deleted — this page records why they were
> switched off and exactly how to bring them back.

## What it was

A desk-claim + agent-messaging layer implemented as four Claude Code hooks:

| hook | event | job |
|---|---|---|
| `office-userprompt.mjs` | UserPromptSubmit | claim a desk, or heartbeat an existing claim |
| `office-pretool.mjs` | PreToolUse (Write/Edit/Bash) | claim before first edit; could **deny** the tool if the desk was occupied |
| `office-inbox.mjs` | UserPromptSubmit | fetch + ack unread messages, inject into context |
| `office-stop.mjs` | Stop | release the desk claim |

All four keyed off one cache file, `~/.claude/office-state-<cwdHash>.json`,
which only `office-userprompt.mjs` could create — and only after a successful
claim against `/api/office/claim` in mooniex-webapp.

## Why it was switched off

The claim gate requires the cwd to resolve to a desk: a
`mooniex-webapp (Desk-X)` folder, a `.desk` marker file, or the `MOONIEX_DESK`
env var. **The Desk pattern was retired 2026-05-28 by ADR-0007** (one repo =
one folder, parallel work via branches + GitHub issues). Once the desk folders
went away, nothing could satisfy the gate, so every hook no-op'd on every turn.

Measured 2026-08-07 before removal:
- **Last successful desk claim: 2026-05-18** — 81 days earlier, and 10 days
  before ADR-0007 landed. The timeline fits exactly.
- Zero `Desk-*` folders anywhere under `Projects/` or `WarpClip Projects/`.
- Zero `.desk` marker files.
- `MOONIEX_DESK` unset.
- No office cache for any current working directory.

Superseded by the `mooniex-coord` MCP server (`send_message`, `read_inbox`,
`list_active_agents`, `claim_message`), which does the messaging half without
the desk-folder coupling.

**Note for anyone reading the 2026-08-06 architecture audit:** that document's
finding W12 claimed these hooks fired two redundant API round-trips per turn.
That was wrong — they fired *nothing*. Both the audit and the correction are
recorded in `org:reference/2026-08-06-agents-system-audit.md`.

## Cost of leaving them wired

Four `node` process spawns per turn that read a nonexistent cache file and
exit. Cheap individually, pure waste collectively, and — for
`office-pretool.mjs` specifically — a hook holding **deny** authority over
every Write/Edit/Bash call while being dead code, which is the part actually
worth removing rather than tolerating.

## How to revive

Nothing was deleted. To bring it back:

1. Restore the hook entries into `~/.claude/settings.json` from the backup at
   `~/.config/mooniex/settings.json.bak-pre-office-removal-20260807121323`, or
   re-add them by hand (UserPromptSubmit ×2, PreToolUse `Write|Edit|Bash` ×1,
   Stop ×1).
2. Give the claim gate something to match — set `MOONIEX_DESK=Desk-CTO` in the
   shell rc, drop a `.desk` file, or restore desk-named folders. **Reviving the
   folder form means reversing ADR-0007** — decide that deliberately.
3. Check `~/.claude/office.json` still holds a valid API token (the one present
   at removal dated from 2026-04-26 and was never re-verified).
4. Confirm the `/api/office/*` endpoints in mooniex-webapp are still live.

Before reviving, decide what it buys over `mooniex-coord` — the MCP path
already covers messaging, so the only unique capability here is desk-level
mutual exclusion, which the branch + GitHub-issue workflow replaced.
