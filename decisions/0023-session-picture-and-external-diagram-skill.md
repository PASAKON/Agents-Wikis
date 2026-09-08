# ADR 0023 — `/session-worktree` ends with a session map; `diagram-design` is an external skill

- **Date:** 2026-09-08, amended 2026-09-09 after two rounds of CEO design questions
- **Status:** Accepted
- **Decider:** CEO; design and build by CTO, session cto-576f0aff
- **Related:** [ADR 0018](0018-skill-lifecycle-and-curator.md) (external skills are symlinks the curator never touches), IRON §32 (tab as live status), IRON §35 (one session, one problem; park the rest)

## Context

`/session-worktree` printed two text views — a 🌳 tree for the CTO and a 📋 recap for the CEO. The CEO wants the whole session as **one picture**, and said what it must show (2026-09-09): "ซ้ายคือเริ่มต้น session — CEO ต้องการอะไร → มี task อะไรบ้าง ในแต่ละ block มี task ย่อย 1 2 3 4 5 แต่ละอันเสร็จหรือยัง → block ต่อไปติด task ก่อนหน้า … goal อาจต่อกันหรือแยกกันคนละเส้น … บอกได้ว่าตอนนี้อยู่จุดไหน ออกนอกเส้นทางไปทางไหน" — and asked for questions first, a design meant to last, and **token economy as the first constraint**.

[cathrynlavery/diagram-design](https://github.com/cathrynlavery/diagram-design) (MIT, v2.6.17) is an opinionated editorial diagram system: tokens, 4px grid, mandatory orthogonal connectors, accessible-SVG contract, its own validator. A trial swimlane passed its validator and looked right.

## Decisions

1. **`diagram-design` is installed as an external skill, not vendored.** Clone at `/Users/gob/Projects/external/diagram-design`, **pinned at sha 2724fd2 (v2.6.17)**; symlink `~/.claude/skills/diagram-design → <clone>/skills/diagram-design`. Same pattern as 9arm / wondelai; the curator refuses symlinks (ADR 0018 §4.3). Updates are deliberate (`git fetch && git checkout <sha>`), never tracking `main`. Default skin, on purpose (CEO 2026-09-09); a MoonieX token profile is a separate decision. `~/.claude/CLAUDE.md` routes standalone diagram deliverables here; Mermaid is not a shipped visual.

2. **The model writes facts, a script owns geometry and time.** `tools/session_diagram.py` keeps **one JSON per session** (`state/session-diagrams/<session>.json`, gitignored) and renders it in the diagram-design system. Vocabulary (answers to the design questions):
   - **Goal** = one thing the CEO asked for (เรื่องที่ CEO สั่ง); the charter's Entry Problem is usually G1; DoD items are its tasks. `depends_on` = must follow another goal (arrow); none = a separate line (own row).
   - **Task** = a step with a checkbox, a work-type tag, evidence when done, a reason when blocked.
   - **Detour** = work that left the path: `interrupt` (had to be done, then back on the line — drawn with a return arrow) or `parked` (raised, not done here, sent to LungNote — drawn to a dead end ⊥).
   - **here** = the task/goal being worked on (`◀ HERE`, orange focal block).
   - **Time** = start→finish + minutes per block, **stamped by the script** on status transitions (doing/blocked start the clock, done stops it; items already done when the map is created stay timeless rather than pretending 0 min). Goal times derive from tasks.

3. **Created only at the first `/session-worktree`, then patched.** Never at `/session-open`, never unasked (HARD rule in the skill). Later runs send a **delta** (`{"set": {"G2.3": "done"}, "here": "G2.4", "add": …}` — 1–3 lines); the script merges, stamps, re-renders and the Artifact tool republishes the **same URL** (stable basename = session id). This is where the tokens are saved: no retyped tree, no resent map, no image ever read back by the model.

4. **Delivery = one Artifact link, last line of the reply.** The CEO's answer to "why SomPong, why a PNG": the link opens on any device, zooms crisply, stays private until shared, and never lands in the secretary's chat. One page holds two maps — wide (desktop, left→right) and narrow (phone, stacked) — switched by a media query. Chat output shrinks to the script's status block (📊 counts · 📍 here · 🔴 blockers) + a 5–8 line plain recap + the link. `show --tree` prints the old text tree only when asked. Telegram (`--png --send`) stays an opt-in.

5. **Labels in English, content as typed** (chips, legend, START/DETOUR/PARKED) — the design system's mono uppercase register; Thai node text uses Noto Sans/Serif Thai at ≥10px.

## Consequences

- `/session-worktree` is a three-part answer now; the full tree is gone from the chat by default.
- Every session that asks for a worktree gets one durable map file + one Artifact URL; `session-save`/`merge` can read the JSON later (not wired yet).
- Contabo needs the clone + symlink (and a Chromium only if PNGs are wanted there) — parked, LungNote 0c2b506c.
- Observed during the build: the Agents `main` working tree is shared by every C-level session; another session's `merge_task` parks (stashes `-u`) foreign uncommitted WIP around its merge and restores it about a minute later. Commit early or bind a session worktree (`/session-open` step 4).

## Verification

- `scripts/test_session_diagram.py` — 13 tests: files + status text, accessible-SVG contract on both maps + upstream `self_check.py`, 4px grid, patch merge + auto-stamping, status inference + here auto-advance, refs/cycles/errors, both detour kinds, escaping/truncation, tree + status text, artifact body, Telegram size cap, CLI map→patch→show. Green under the `__main__` harness and pytest.
- First real map: session cto-576f0aff, 2026-09-09 (this session's own history: 4 goals, 12 tasks, 3 detours), `check: OK`, published as the session's Artifact.
- Commits on Agents `main`: 1ca64bd (v1, Telegram PNG), c3d5d2b (v2, Artifact link), and the v3 session-map commit of 2026-09-09.
