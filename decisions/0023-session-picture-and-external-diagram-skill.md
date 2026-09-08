# ADR 0023 — `/session-worktree` ends with a picture; `diagram-design` is an external skill

- **Date:** 2026-09-08
- **Status:** Accepted — CEO ask 2026-09-08: "เอามาใช้เป็น Skill ได้ไหม ใน session-worktree อยากให้ทำเป็น diagram ส่งมาด้วย ในบรรทัดสุดท้าย … เราจะได้เห็นภาพรวมของทั้ง session เป็น diagram เลย"
- **Decider:** CEO; design and build by CTO, session cto-576f0aff
- **Related:** [ADR 0018](0018-skill-lifecycle-and-curator.md) (external skills are symlinks the curator never touches), IRON §32 (tab as live status), CEO order #38 (media to the CEO as a real file)

## Context

`/session-worktree` printed two text views — the 🌳 tree for the CTO and the 📋 plain recap for the CEO. The CEO wants the whole session as **one picture on the phone**.

The CEO pointed at [cathrynlavery/diagram-design](https://github.com/cathrynlavery/diagram-design) (MIT, 34k★, v2.6.17): an opinionated editorial diagram system — 39 visual types, a skinnable token set (`references/style-guide.md`), a 4px grid, mandatory orthogonal r=8 connectors, an accessible-SVG contract, and its own validator `scripts/self_check.py`. Output is a single self-contained HTML file (inline SVG + CSS, Google Fonts only). A trial diagram — the org task lifecycle as a swimlane — passed its validator and rendered correctly, so the system is real, not a prompt.

## Decision

1. **Install `diagram-design` as an external skill, not vendored into the repo.** Clone under `/Users/gob/Projects/external/diagram-design`, **pinned at sha 2724fd2 (v2.6.17)**; symlink `~/.claude/skills/diagram-design → <clone>/skills/diagram-design`. Same pattern as 9arm / wondelai; the curator refuses symlinks (ADR 0018 §4.3), so it can never patch upstream. Updates are deliberate: `git fetch && git checkout <sha>` — never track `main`. The skill's first-run "customise the style guide?" gate is answered here: **default skin, on purpose**; a MoonieX token profile is a separate, CEO-facing decision.
2. **The model writes facts, a script owns geometry.** `tools/session_diagram.py` takes the session tree as JSON on stdin (entry problem, DoD, nodes with `done|doing|todo|blocked` + work-type tag + evidence + `blocked_on`) and renders a vertical tree in the diagram-design system: node treatment from the SKILL.md §5 table plus the kanban card states (done = store, doing = focal + `◀ HERE`, todo = optional dashed, blocked = accent bar + reason), Instrument Serif / Geist / Geist Mono with Noto Thai for Thai text, legend strip + counts, `<title>/<desc>` contract. `--print-tree` emits the 🌳 text in the skill's exact format, so view 1 and the picture come from one source and cannot disagree. A hand-drawn diagram costs ~15 minutes of SVG per picture; the script costs 5 seconds.
3. **Delivery is a Telegram photo** through `lib.telegram_out.send_media_to_ceo` (`--send`): a real file, never a link (CEO order #38). The chat stays text (memory `cto-chat-text-output`); the **last line** of the worktree is the PNG path plus the script's own `telegram:` verdict — `ok` is the only thing that counts as sent.
4. **PNG via the Chrome CLI, not Playwright.** Playwright is not installed and upstream's `export.md` says not to auto-install it. Headless Chrome `--screenshot` renders the fonts correctly but never exits on macOS (its updater child) — the script polls for the file, then terminates Chrome. Scale is lowered automatically so Telegram's `sendPhoto` cap (width + height ≤ 10000 px) holds.
5. **Where Chrome is absent** (Contabo today), the script skips the PNG and sends the HTML as a document, and the skill says so plainly.

## Consequences

- `/session-worktree` gains view 3; `state/session-diagrams/` holds the JSON/HTML/PNG per run and is gitignored.
- Contabo sessions need the same clone + symlink (and a Chromium) before they produce PNGs — follow-up, not blocking.
- `~/.claude/CLAUDE.md` routes any standalone diagram deliverable to `diagram-design`; `artifact-diagramming` stays for figures inside an Artifact page; Mermaid is not a shipped visual.
- Observed during the build: the Agents `main` working tree is shared by every C-level session. Another session's `merge_task` parks (stashes, `-u`) a foreign session's uncommitted WIP around its merge and restores it afterwards — the tree flickers for about a minute. Commit early, or bind a session worktree (`/session-open` step 4).

## Verification

- `scripts/test_session_diagram.py` — 10 tests: accessible-SVG contract, upstream `self_check.py`, 4px grid, the four states + counts, escaping/truncation, tree text format, Telegram size cap, CLI. Green under both the `__main__` harness and pytest.
- First real run: session cto-576f0aff, 2026-09-08 23:02 — `check: OK`, `png: … (scale 2)`, `telegram: ok`.
- Commit on Agents `main` 2026-09-08: `tools/session_diagram.py`, `scripts/test_session_diagram.py`, `.claude/skills/session-worktree/SKILL.md`, `.gitignore`.
