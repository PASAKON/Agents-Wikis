# ADR 0023 — `/session-worktree` ends with a session map; `diagram-design` is an external skill

- **Date:** 2026-09-08, amended 2026-09-09 (twice) after four rounds of CEO design questions
- **Status:** Accepted
- **Decider:** CEO; design and build by CTO, session cto-576f0aff
- **Related:** [ADR 0018](0018-skill-lifecycle-and-curator.md) (external skills are symlinks the curator never touches), IRON §32 (tab as live status), IRON §35 (one session, one problem; park the rest), the standard itself: [playbooks/session-map.md](../playbooks/session-map.md)

## Context

`/session-worktree` printed two text views — a 🌳 tree for the CTO and a 📋 recap for the CEO. The CEO wants the whole session as **one picture**, and said what it must show (2026-09-09): "ซ้ายคือเริ่มต้น session — CEO ต้องการอะไร → มี task อะไรบ้าง ในแต่ละ block มี task ย่อย 1 2 3 4 5 แต่ละอันเสร็จหรือยัง → block ต่อไปติด task ก่อนหน้า … goal อาจต่อกันหรือแยกกันคนละเส้น … บอกได้ว่าตอนนี้อยู่จุดไหน ออกนอกเส้นทางไปทางไหน" — and asked for questions first, a design meant to last, and **token economy as the first constraint**. Later the same night: a flag on the last block, status colours, icons instead of emoji, and the delegated workers on the map — "ถ้าได้มาตรฐานแล้ว จะได้ update เป็น skill ใส่ไว้ CTO คนอื่น ๆ จะได้ทำออกมาโดยใช้ template เดียวกัน".

[cathrynlavery/diagram-design](https://github.com/cathrynlavery/diagram-design) (MIT, v2.6.17) is an opinionated editorial diagram system: tokens, 4px grid, mandatory orthogonal connectors, accessible-SVG contract, its own validator. A trial swimlane passed its validator and looked right.

## Decisions

1. **`diagram-design` is installed as an external skill, not vendored.** Clone at `/Users/gob/Projects/external/diagram-design`, **pinned at sha 2724fd2 (v2.6.17)**; symlink `~/.claude/skills/diagram-design → <clone>/skills/diagram-design`. Same pattern as 9arm / wondelai; the curator refuses symlinks (ADR 0018 §4.3). Updates are deliberate (`git fetch && git checkout <sha>`), never tracking `main`. `~/.claude/CLAUDE.md` routes standalone diagram deliverables here; Mermaid is not a shipped visual.

2. **The model writes facts, a script owns geometry and time.** `tools/session_diagram.py` keeps **one JSON per session** (`state/session-diagrams/<session>.json`, gitignored) and renders it. Vocabulary (answers to the design questions):
   - **Goal** = one thing the CEO asked for (เรื่องที่ CEO สั่ง); the charter's Entry Problem is usually G1. `depends_on` = must follow another goal (arrow); none = a separate line (own row).
   - **Task** = a step with a status icon, a work-type tag, evidence when done, a reason when blocked.
   - **Detour** = work that left the path: `interrupt` (had to be done, then back on the line — drawn with a return arrow) or `parked` (raised, not done here, sent to LungNote — drawn to a dead end ⊥).
   - **here** = the task/goal being worked on (map-pin, amber, 2px outline).
   - **Time** = start→finish + minutes per block, **stamped by the script** on status transitions (doing/blocked start the clock, done stops it; items already done when the map is created stay timeless). Goal times derive from tasks; FINISH shows the session total.

3. **Created only at the first `/session-worktree`, then patched.** Never at `/session-open`, never unasked (HARD rule in the skill). Later runs send a **delta** (`{"set": {"G2.3": "done"}, "here": "G2.4", "add": …}` — 1–3 lines); the script merges, stamps, re-renders and the Artifact tool republishes the **same URL** (stable basename = session id). This is where the tokens are saved: no retyped tree, no resent map, no image ever read back by the model.

4. **Delivery = one Artifact link, last line of the reply.** The CEO's answer to "why SomPong, why a PNG": the link opens on any device, zooms crisply, stays private until shared, and never lands in the secretary's chat. **One layout only — horizontal, left→right** (CEO: "แนวนอนอย่างเดียว"); a phone scrolls the map sideways inside its own frame; a responsive stacked variant was built and removed the same night. Chat output shrinks to the script's status block (📊 counts · 📍 here · 🔴 blockers · 🤖 workers) + a 5–8 line plain recap + the link. `show --tree` prints the old text tree only when asked. Telegram (`--png --send`) stays an opt-in.

5. **Labels in English, content as typed** (chips, legend, START/FINISH/DETOUR/PARKED/WORKER/HERE) — the design system's mono uppercase register; Thai node text uses Noto Sans/Serif Thai at ≥10px.

6. **The visual standard (CEO 2026-09-09, second round):**
   - **FINISH block** at the far right, under a flag: the charter's Definition of Done as a checklist (`F.1…F.n`, ticked via `set`), the session's total time; every line runs into it; ink outline until every goal and every DoD item is done, then green.
   - **Status colours** on top of diagram-design's paper/ink: done green `#3a8f5c`, doing/here amber `#d4952b`, blocked red `#c9452e`, not started grey dashed, START/FINISH ink, detour blue `#2e5aa8`, parked soft grey, worker teal `#1f8a8a`, arrows muted. ~10% tints for fills, full colour for strokes/chips/icons. **Never colour alone** — every state carries an icon and a word (greyscale- and colour-blind-safe).
   - **Icons, not emoji, on the map** — Tabler Icons (MIT, the same source as diagram-design's icon primitive), embedded as `<symbol>`s: check · clock · circle-dashed · alert-triangle · map-pin · flag · player-play · arrow-back-up · parking · robot, plus one per work type. Emoji stay in the terminal chat lines.
   - **Workers on the map**: every task this session delegated (`tasks.owner_cto` = session) is read live from `state/tasks.db` at each render — role, task id, tmux name, worker session, host, status, runtime — and drawn as a teal card under the goal it serves (the CTO assigns once with `"workers": {"G2": ["task-…"]}`); unassigned ones sit in a band under the rows; finished ones stay, green or grey.
   - **One template for every C-level**: the standard is written once in `playbooks/session-map.md` (anatomy, colours, icons, vocabulary, data contract), implemented once in the script, described once in the skill. Per-session restyling is not a thing.

## Consequences

- `/session-worktree` is a three-part answer now; the full tree is gone from the chat by default.
- Every session that asks for a worktree gets one durable map file + one Artifact URL; `session-save`/`merge` can read the JSON later (not wired yet).
- Contabo needs the clone + symlink (and a Chromium only if PNGs are wanted there) — parked, LungNote 0c2b506c.
- Observed during the build: the Agents `main` working tree is shared by every C-level session; another session's `merge_task` parks (stashes `-u`) foreign uncommitted WIP around its merge and restores it about a minute later. Commit early or bind a session worktree (`/session-open` step 4).

## Verification

- `scripts/test_session_diagram.py` — 16 tests: files + status text, accessible-SVG contract + upstream `self_check.py`, 4px grid, patch merge + auto-stamping (incl. DoD and worker assignment), status inference + here auto-advance + finished(), refs/cycles/errors, both detour kinds, the FINISH block (flag, DoD rows, green when complete), workers (under a goal, the bottom band, live counts, tree/status text), icons + palette, escaping/truncation, tree + status text, artifact body, Telegram size cap, CLI map→patch→show. Green under the `__main__` harness and pytest.
- Real map: session cto-576f0aff (this build's own history), `check: OK`, one Artifact URL republished on every run.
- Commits on Agents `main`: 1ca64bd (v1, Telegram PNG), c3d5d2b (v2, Artifact link), 2cf76ba (v3, the session map), 4e69ad9 (horizontal only), plus the v4 visual-standard commit of 2026-09-09.
