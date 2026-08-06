# Agents System Architecture Audit — 2026-08-06

**Scope:** `/Users/gob/Projects/Agents` (org runtime meta-repo) — engine code, scripts/tools/roles, disk footprint, wiki+skills+hooks context-loading. **Method:** 4 parallel read-only Explore agents (core engine / scripts+tools+roles / disk-bloat root-cause / wiki+skills+hooks token footprint), ground-truth verified (file reads, `git log`, `du`, `wc`), not derived from memory. **By:** CTO, requested by CEO. **Not yet actioned** — this is the diagnostic; CEO picks what to execute next.

## TL;DR
- Core engine has real defense-in-depth where it got fixed (merge-safety, terminal-status races) — but **three parallel, drifting tool-implementation surfaces** (MCP subprocess vs in-process SDK vs REPL) is the single biggest architecture debt: the CEO-facing `agents-chat` REPL is literally missing `revert_task_tool`.
- **~5GB of the 5.3GB in `worktrees/`** is stale/dormant (terminal-status or untouched 5-10+ weeks) because GC never sweeps worktrees for cancelled/stalled/failed tasks — narrow, fixable gap, biggest disk win available.
- Debug/perf-check layer (watchdog, GC, 27 test files, cost-guardian) is **already proportionate, mostly silent** — not the bloat source. The one thing wrong with it is it doesn't do worktree cleanup, i.e. it under-reaches, doesn't over-reach.
- The single biggest **token** lever isn't in this repo at all — it's ~150-200+ irrelevant `ecc:*` domain skill packs (kotlin/cpp/rust/healthcare/logistics/etc., already marked "hard skip" in CLAUDE.md) that still get fully enumerated into every session's context, because CLAUDE.md can tell Claude not to *invoke* them, but can't stop the harness from *listing* them.
- Real duplicate-file clusters exist (4 near-identical FB poster scripts, 6 near-identical scene-gen pipelines, 2 dead one-campaign clusters) — genuine, safe consolidation targets, exactly what was asked.

## 1. Strengths — keep

| Area | Evidence | Why it's good |
|---|---|---|
| Terminal-status race guard | `lib/db.py:326-389` | Folds "don't resurrect done/merged" into the UPDATE's WHERE clause, not read-then-write. Has a regression test (`scripts/test_status_terminal_guard.py`). |
| merge_task no-op protection | `tools/git_ops.py:178-283` | 4 independent guards (dirty-base preflight, ancestor precheck, post-merge SHA-advance verify, conflict-vs-empty disambiguation), each comment cites the real incident it closed (#13/#29/#42). The org's own past "merge_task can report merged=true but no-op" bug is now genuinely fixed, with defense in depth. |
| recall()/reflect() | `lib/recall.py`, `lib/reflect.py` | Deliberately reject embeddings ("keyword ranking over SQLite is enough at this scale") — read-only, can't corrupt state. Good judgment restraint, not under-built. |
| TOON encoder | `lib/toon.py`, ADR-0010 | Real, shipped, measured token savings — proof the org already knows how to do token-efficiency right. |
| Watchdog pid-gated tab closure | `runners/watchdog.py:157-170` | Refuses to kill a tab unless the pid was stamped by the delegate-spawn path — won't auto-kill a manually-attached human session. |
| Worktree cleanup trigger point | `tools/git_ops.py:336` inside `merge_task()` | Correctly placed at the one guaranteed-safe moment (after a merge that actually advanced the base). Just too narrow in scope (see Weakness #5). |
| Wiki split (ADR-0013) | `org:IRON-RULES.md` (993 lines) + `mooniex:IRON-RULES.md` (1,524 lines) | Zero overlapping section numbers across the two files — the split is actually coherent. |
| ADR discipline | 16 files in `org:decisions/`, 30 in `mooniex:decisions/` | Real decisions get written down with dates/attribution, not just made verbally. |
| Test coverage for org tooling | 27 `test_*.py` under `scripts/` | Not zero — spawn routing, tab routing, owner_cto routing, iTerm typewriter (47/47 green) all have regression tests. |
| `.gitignore` core coverage | repo root | `worktrees/`, `.venv/`, `__pycache__/`, `*.pyc` correctly excluded — the gaps are narrower than "gitignore is broken" (see Weakness #9). |

## 2. Weaknesses — ranked

**Critical**

1. **Three drifting orchestration-tool implementations.** `runners/cto_mcp_server.py` (17 tools, real prod path via `cto-claude.sh`) vs `runners/cto.py` (14 tools, used by `main.py`) vs `runners/cto_chat.py` (imports 13, hardcodes the literal string `"Tools: 13"`). The live `agents-chat` REPL alias is missing `check_collisions`, `recall`, `reflect`, **and `revert_task_tool`** entirely. *Failure:* CEO runs `agents-chat`, asks CTO to revert a bad merge — the tool isn't on that code path.
2. **owner_cto cross-CTO-window guard exists on only one of the three paths.** `cto_mcp_server.py:42-59` blocks a second CTO window from mutating another's task; `cto.py`'s `t_delegate`/`t_merge`/`t_reopen` (`:138-140,182-184,192-204`) call the underlying ops with **no ownership check**, and `cto_chat.py` inherits the gap. Same class of risk as the LungNote "owner_cto claim discipline" hardening note (flagged 2026-06-15, deprioritized as "optional, not urgent") — but wider than that framing suggested: it's not an edge case, it's the entire SDK-based path (one-shot `main.py` + the interactive REPL), unguarded by design.
3. **`depends_on` is invisible to the lock/collision layer.** `db.find_conflicts` (`lib/db.py:511-544`) and `delegate_task`'s pre-flight (`tools/delegate.py:412-425`) do pure touches-intersection with no `depends_on` awareness. The one function that does honor it, `db.pending_tasks_for` (`lib/db.py:444-459`), has **zero callers**. This is exactly the bug the org's own memory already works around by hand ("lock layer ignores depends_on; never pre-queue dependent waves w/ overlapping touches; serialize") — confirmed still unfixed at the code level, only mitigated by CTO discipline.

**High**

4. **Retry-before-claim double-spawn window still open** (`tools/delegate.py:438-453`) — matches the existing "delegate retry double-spawn" memory; DB-layer `claim_task` prevents two DEVs actually *running*, but doesn't stop the wasted second spawn attempt, which still relies on a racy AppleScript title-scan.
5. **Worktree GC gap — the actual source of the 5.3GB.** `gc_stale_tasks.py` cancels stale tasks and marks agents `stalled`, but only calls `db.update_status()`/`release_task_locks()` — **never `remove_worktree`**. Any task ending `cancelled`/`stalled`/`failed` leaks its worktree forever; anything sitting in `review` is never revisited. Measured: of 25 worktree dirs on disk, ~20 (~5.0GB) are terminal-status or dormant ≥5 weeks (12 of them dormant 10+ weeks); only 5 (371MB) show activity in the last week.
6. One worktree (`task-df93e0ed`) sits at DB status `merged`, but `merge_task()`'s only terminal write is `done` — that row reached `merged` via some other path this audit didn't trace, and never got cleaned up. Worth a 5-minute look, not urgent.

**Medium**

7. Dead code: `runners/dev.py` (148 lines, zero callers, superseded by `dev_init.py` per that file's own docstring), `db.is_session_busy` (`lib/db.py:609-617`, zero callers — `tools/itermtab.py:240-250` reimplements the same check inline instead of calling it).
8. Schema drift: `state/tasks.db` has a `claudesign_project_id` column absent from `lib/db.py`'s migration/valid-column lists, 100% NULL, zero code references — added out-of-band, wouldn't exist on a fresh `db.init()`.
9. **37% of `scripts/` (47/126 files) is untracked in git — including live, load-bearing files.** `hook-skill-log.py` and `hook-skill-suggest.py` are both untracked *and* directly referenced by `.claude/settings.json`. Same pattern one level up: **91 of the repo's 104 local Claude skills are symlinks into `.agents/skills/`, and `.agents/` itself is fully untracked** (3.3MB, last touched 2026-07-27). None of this is dead — it's live and load-bearing — which makes it worse, not better: a fresh clone of this repo silently loses working hooks and 91 skills, with no error and no git history to recover from.
10. `.asset-cache/` — 72MB, Kenney game-asset kits + a full Godot road-generator addon, zero references anywhere in the repo. Leftover from Mooniex Video Engine (Godot), which the org's own memory records as closed 2026-06-23 (pivoted to web-2D). 6+ weeks dead.
11. `run-fomo-scanner.sh` + `com.mooniex.fomo-scanner.plist` document an "every 4h" scan, but the plist is **not installed** under `~/Library/LaunchAgents/` — the automation is documented as running and isn't. A believed-alive automation that's actually silent is exactly the kind of gap a debug/perf layer should catch and doesn't.
12. Two hooks independently reimplement the same desk-claim/heartbeat cycle: `office-userprompt.mjs` (UserPromptSubmit) and `office-pretool.mjs` (PreToolUse) both hit `/api/office/claim`+`heartbeat` — a turn with both a prompt and a tool edit fires the same network round-trip twice.
13. Stale/misleading docs found in passing: `config/wikis.yaml:11-13` claims the `org:` wiki root "doesn't exist on disk yet" — false, it's a live, independently-committed repo with a commit today. `office-pretool.mjs:17` and `office-inbox.mjs:23` cite `LLMs/IRON-RULES.md §1`/`§17` — those sections actually live in `Agents-Wikis/IRON-RULES.md` (org:) now, post ADR-0013 split; citations were never updated. `§18` is undefined in either IRON-RULES.md (likely a harmless renumber, not verified further).
14. `testsprite/` (fully untracked, zero references, 69 days stale — a TestSprite QA plan for mooniex.com) and `demo-api/` (generic BMI-calculator toy, gitignored, 80 days stale, no Mooniex content) — onboarding/scratch experiments never cleaned up.

## 3. Excess — cut or merge (direct answer to "อะไรที่เกินความจำเป็น")

| Item | Disposition | Evidence |
|---|---|---|
| `runners/dev.py` | Delete | Zero callers, confirmed superseded |
| `db.is_session_busy` | Delete | Zero callers, reimplemented inline elsewhere |
| `claudesign_project_id` column | Drop or wire up | 100% NULL, zero references |
| `.asset-cache/` (72MB) | Delete | Dead project (Video Engine, closed 6+ wks) |
| `testsprite/`, `demo-api/` | Delete | Confirmed unused, 69-80 days stale |
| SpaceX cluster — 4 files (`spcx_*.py`) | Archive or delete (commit first — currently untracked, `rm` = unrecoverable) | Single closed campaign, 2026-06-12/13, zero references |
| Frost Knight cluster — 3 files (`frost_knight_*.py`) | Archive or delete (commit first) | Single closed campaign, 2026-06-21/22, zero references |
| 4× `post-*.js` FB publishers | Merge into one shared lib | Self-documented as clones of each other; identical idempotency/path boilerplate ×4 |
| 6× scene-gen+overlay poster scripts (`mooniex_poster.py`, `gen-rebate-scenes.py`, `gen-brandprompt-scenes.py`, `frost_knight_thumbs.py`, `spcx_poster.py`, `spcx_musk_gen.py`) | Extract shared pipeline lib, keep thin per-campaign config | Same recipe (fal scene-gen → Playwright text-overlay → logo stamp) hand-copied 6x |
| `office-userprompt.mjs` + `office-pretool.mjs` | Collapse desk-claim logic into one hook | Confirmed duplicate API calls same turn |
| **Three orchestration tool surfaces** (`cto_mcp_server.py`/`cto.py`/`cto_chat.py`) | Collapse to one source of truth (generate all three call-sites from one tool manifest) | Root cause of Weakness #1 — highest-value structural fix, not a quick cut |

## 4. Debug / perf-check systems — verdict

Inventory: `watchdog.py` (stall detection, ~10min loop) + `gc_stale_tasks.py` (stale-task cancellation) + 27 `test_*.py` files + `hook-skill-log.py` (silent TSV log) + `session-deadline-check.py` (LungNote surfacing) + cost-guardian plugin (budget-guard PreToolUse + track-usage PostToolUse, SQLite log).

**Verdict: proportionate, mostly silent, not the bloat source.** Almost every hook in this list either prints nothing (logs to a file/DB) or prints ≤1-3 lines. The org's instinct here has been good (see recall/reflect/TOON in Strengths). The one real problem is that this layer **under-reaches**: GC doesn't clean worktrees (→ Weakness #5), and nothing monitors whether `run-fomo-scanner`'s cron is actually installed (→ Weakness #11). The only redundancy found is `gateguard-category` (repo) and `cost-guardian` (plugin) independently gating overlapping tool matchers (Edit/Write/MultiEdit) — minor, not worth urgent action.

## 5. Token footprint — real numbers

| Source | Size | Auto-loaded every session? |
|---|---|---|
| 3× CLAUDE.md (user+workspace+repo) | 285 lines / 16KB | Yes — fixed cost, already lean |
| `memory/MEMORY.md` index | 113 lines / 20KB | Yes — but by design (lazy index, not full recall) |
| `memory/*.md` (113 files) | 3,443 lines / 324KB | **No** — loaded on demand only. Already well-architected; not a live problem. |
| `IRON-RULES.md` ×2 (org+mooniex) | 2,517 lines total | **No** — wiki_read on demand, not system-prompt-injected |
| **Available-skills listing** | ~104 local (Agents repo) + 58 user-level + **229 ecc + 8 finance + 4 cost-guardian + ~6 meigen plugin skills** | **Yes — every session, in full**, regardless of relevance |
| 18 hook scripts across SessionStart/UserPromptSubmit/PreToolUse/PostToolUse/Stop | 1,834 lines combined | Mixed — most silent, a few inject real text every turn (caveman ruleset ~143 lines on SessionStart, office-inbox ≤20-msg block, allowlist-notify block) |

**The lever that actually matters:** this repo's own `CLAUDE.md` already maintains a "Hard skips" list of ~35+ `ecc:*` domain packs (kotlin/cpp/rust/go/swift/flutter/csharp/fsharp/perl/laravel/django/quarkus/springboot/nestjs/angular/nuxt4/dotnet/scientific/healthcare/logistics/network/homelab/defi/etc.) — but that list only stops Claude from *invoking* them. It cannot stop the harness from *enumerating* all ~229 of them into context every session. That enumeration is fixed overhead on every session's first system-reminder block, unrelated to what the session is actually about. **This is the single biggest, most concrete token-reduction opportunity found today** — and it needs a plugin/marketplace-level config change (disable the irrelevant domain packs), not a CLAUDE.md edit, since CLAUDE.md has already done everything it can here.

Second lever: the `office-userprompt.mjs`/`office-pretool.mjs` duplicate desk-claim calls (Weakness #12) — pure waste, zero functional value, easy fix.

Third lever (positive, not a fix): TOON (`lib/toon.py`, ADR-0010) is proven and already shipped for claudeflow's Kimi loop — extending it to CTO-facing high-volume MCP returns (`get_task`, `list_todos`, wiki-search hits, `stats`) would reuse an already-built, already-measured mechanism rather than requiring anything new.

## 6. Skill / rule recommendations

1. **New IRON-RULE**: one orchestration tool-surface per role — ban parallel/drifted tool lists; if in-process-SDK and subprocess-MCP must both exist, generate both from one manifest so they structurally can't drift again. (→ Weakness #1, #2)
2. **Extend `gc_stale_tasks.py`**: terminal-status transition (`done`/`cancelled`/`stalled`/`failed`) must trigger `remove_worktree` in the same pass, plus a re-sweep for anything parked in `review` past N days. (→ Weakness #5 — highest-value, lowest-risk fix from this audit, reclaims ~5GB, prevents recurrence)
3. **New skill or CLAUDE.md rule** — "untracked-but-load-bearing check": before any disaster-recovery drill or new-machine onboarding, grep for files referenced by `settings.json`/`.claude/skills` that are NOT git-tracked. Concrete trap found today: 2 live hooks + 91 skills' backing store. (→ Weakness #9)
4. **Extend `cto-merge-checklist` skill**: if a task reaches `merged` via an unusual status path (not the normal review→merge transition), manually verify worktree cleanup ran. (→ Weakness #6)
5. **Org rule**: no untracked one-off campaign script survives past 14 days after the campaign ends — commit under `scripts/campaigns/<date>-<name>/` or delete. Prevents the SpaceX/Frost-Knight-style cluster from recurring. (→ §3)
6. **Skill-authoring guidance**: before writing a new content-gen script, check for an existing shared pipeline first (scene-gen→overlay, PDF-via-headless-Chrome, FB-post-with-idempotency all already exist 3-6x over). GateGuard's existing Edit/Write fact #2 ("confirm no existing file serves the same purpose") already asks this — it's just not catching this pattern in practice. (→ §3)
7. **Plugin-config action (not a CLAUDE.md rule — needs an actual plugin/marketplace change)**: disable the ~35+ irrelevant `ecc:*` domain skill-packs already listed as "Hard skips," so they stop being enumerated every session instead of just being marked don't-invoke. Biggest concrete token win found today. (→ §5)
8. Collapse `office-userprompt.mjs` + `office-pretool.mjs` into one hook. (→ Weakness #12)
9. Fix the two broken `IRON-RULES §1/§17` citations in `office-pretool.mjs`/`office-inbox.mjs` (leftover from the ADR-0013 wiki split) and the stale "doesn't exist on disk" comment in `config/wikis.yaml:11-13`. Small — can ride along with the already-open "ล้าง wiki ตาม IRON §41" LungNote item (due 2026-08-31) instead of a new task.

---
Not filed as GitHub issues or delegated to DEV yet — pending CEO pick on what to act on.
