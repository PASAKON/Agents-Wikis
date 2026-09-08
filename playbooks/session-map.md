# Session Map — the standard every C-level session draws the same way

The picture behind `/session-worktree`. One map per session, one link, redrawn by
`tools/session_diagram.py` from one JSON that the C-level patches with only what changed.
Designed with the CEO on 2026-09-09 (two rounds of questions; [ADR 0023](../decisions/0023-session-picture-and-external-diagram-skill.md)).
Change the standard here first, then in the script — never per session.

## What it shows, left → right

```
START ──▶ [G1 ✓✓✓ 22:39→23:04·25m] ──▶ [G2 ✓○○ ◉ HERE] ──▶ [G4 ○○] ──▶ ⚑ FINISH (Definition of Done)
  └─────▶ [G3 ✓○ ▲ blocked] ──────────────────────────────────────────┘
               ╎🤖 worker task-…          ╎↩ detour that came back      ╎⊥ parked → LungNote
```

- **START** — what the CEO asked for, in their words.
- **Goal block** — one thing the CEO asked for (เรื่องที่ CEO สั่ง). A goal that needs an earlier goal follows it with an arrow (`depends_on`); a goal on its own line sits on its own row. Inside: the tasks 1-2-3 with a status icon each, the block's `start→finish · minutes`, evidence on done tasks, the reason on blocked ones.
- **FINISH** — the session's Definition of Done under a flag; every line runs into it; turns green when every goal and every DoD item is done. Shows the session's total time.
- **HERE** — the task (or goal) being worked on: amber, 2px outline, a map-pin.
- **Worker** — a task this session delegated, hanging under the goal it serves: `role · task id · status` chips, the worker's task title, then `task-id · wd-tmux · session · host · runtime`. Read live from `state/tasks.db` (`owner_cto` = this session) at every render — nothing typed. Workers not yet tied to a goal sit in a band under the rows.
- **Detour** — work that left the path. `interrupt` = had to be done, then back on the line (down and back up); `parked` = raised, not done here, sent to LungNote (down to a dead end ⊥).

## Colours — status, never decoration

Fills are ~10% tints; strokes, chips and icons carry the full colour. **Never colour alone** — every state also has an icon and a word, so the map reads in greyscale and for colour-blind readers.

| State | Colour | Fill | Extra |
|---|---|---|---|
| Done | green `#3a8f5c` | `rgba(58,143,92,.10)` | check icon |
| Doing · Here | amber `#d4952b` | `rgba(212,149,43,.12)` | clock icon; HERE adds a 2px outline + map-pin |
| Blocked / failed | red `#c9452e` | `rgba(201,69,46,.10)` | alert icon, 4px red bar on the block's left edge, reason line |
| Not started | grey, dashed `4,3` | `rgba(45,49,66,.02)` | dashed-circle icon |
| START / FINISH | ink `#2d3142` | white | play / flag icon; FINISH goes green when complete |
| Detour (came back) | blue `#2e5aa8` | `rgba(46,90,168,.08)` | return-arrow icon; green once it came back |
| Parked | soft grey `#7a8399`, dashed | `rgba(45,49,66,.02)` | parking icon, ⊥ dead-end stub |
| Worker | teal `#1f8a8a` | `rgba(31,138,138,.10)` | robot icon; done → green, failed → red, cancelled → grey dashed |
| Arrows | muted `#4f5d75` | — | "needs the previous goal"; dashed = detour / worker branch |

Paper `#f5f5f5`, ink `#2d3142`, muted `#4f5d75`, soft `#7a8399` are diagram-design's defaults — the status palette sits on top of that skin, not instead of it. Type: Instrument Serif (headline), Geist (names), Geist Mono (chips, ids, times), Noto Thai for Thai text.

## Icons — Tabler Icons (MIT), outline, never emoji on the map

| Meaning | Icon | Meaning | Icon |
|---|---|---|---|
| done | `check` | READ / RECON | `search` |
| doing | `clock` | RESEARCH | `book` |
| not started | `circle-dashed` | ANALYZE / DECIDE | `bulb` |
| blocked | `alert-triangle` | BUILD | `hammer` |
| here | `map-pin` | FIX | `tool` |
| start | `player-play` | DESIGN | `palette` |
| finish | `flag` | SETUP | `settings` |
| detour (returned) | `arrow-back-up` | TEST / VERIFY | `test-pipe` |
| parked | `parking` | SHIP | `rocket` |
| worker | `robot` | DOC | `file-text` |

Chips are rectangles (`rx=2`), never pills; labels are English mono uppercase (DONE, DOING, BLOCKED, START, FINISH, DETOUR, PARKED, WORKER, HERE); the content — goal and task titles — is whatever language it was typed in. Emoji stay in the **chat** status lines (📊 📍 🔴 🤖 ⏱), where a terminal cannot draw icons.

## Data contract

One file: `state/session-diagrams/<session>.json` (gitignored). Refs: `G2` goal · `G2.3` task · `D1` detour · `F.2` DoD item.

```json
{"entry_problem": "one sentence", "start": "what the CEO asked",
 "dod": [{"text": "…", "done": false}],
 "goals": [{"id": "G1", "title": "…", "type": "BUILD", "depends_on": [],
            "tasks": [{"title": "…", "status": "done|doing|todo|blocked", "type": "FIX", "evidence": "sha", "blocked_on": "…"}],
            "detours": [{"title": "…", "kind": "interrupt|parked", "status": "done", "note": "LungNote id"}]}],
 "workers": {"G1": ["task-149e6c86"]},
 "here": "G1.2"}
```

Patch (every later run): `{"set": {"G1.2": "done", "F.1": "done"}, "evidence": {"G1.2": "sha"}, "here": "G1.3",
"add": {"tasks": {"G2": [{"title": "…"}]}, "goals": […], "detours": {"G1": […]}, "dod": […]},
"blocked": {"G2.1": "รอ …"}, "workers": {"G2": ["task-…"]}}` — 1–3 lines is normal.

Times are stamped by the script when a status changes (doing/blocked start the clock, done stops it; items already done at map creation stay timeless). Goal times derive from tasks; the FINISH block shows the session total.

## Rules

1. The map exists only after the CEO asks for a worktree — never at `/session-open`, never unasked.
2. Patch, don't resend. Same URL every run (Artifact republish of `<session>.artifact.html`).
3. Horizontal only; a phone scrolls sideways. One layout, everywhere.
4. Chat = the script's status block + a 5–8 line plain recap + the link. The full tree only via `show --tree`.
5. `check: OK` (diagram-design's `self_check.py`) before publishing.
6. This page is the standard: colours, icons and vocabulary change here first, then in `tools/session_diagram.py`, then every session inherits it. Per-session restyling is not a thing.

## Where

- Script: `tools/session_diagram.py` (map / patch / show / render / sample) · tests: `scripts/test_session_diagram.py`
- Skill: `.claude/skills/session-worktree/SKILL.md`
- Skin source: `~/.claude/skills/diagram-design` (external, pinned v2.6.17)
