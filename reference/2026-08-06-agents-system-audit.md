# Agents System Architecture Audit — 2026-08-06

**Scope:** `/Users/gob/Projects/Agents` (org runtime meta-repo) — engine code, scripts/tools/roles, disk footprint, wiki+skills+hooks context-loading. **Method:** 4 parallel read-only Explore agents, ground-truth verified (file reads, `git log`, `du`, `wc`). **By:** CTO, requested by CEO.

> **⚠️ READ THE STATUS SECTION AT THE BOTTOM FIRST.** Remediation is complete as of 2026-08-07, and **three of this document's original findings turned out to be factually wrong** when acted on. The corrections are recorded at the end — don't act on the raw findings below without checking there.

## TL;DR (as originally written, 2026-08-06)
- Core engine has real defense-in-depth where it got fixed (merge-safety, terminal-status races) — but **three parallel, drifting tool-implementation surfaces** (MCP subprocess vs in-process SDK vs REPL) is the single biggest architecture debt: the CEO-facing `agents-chat` REPL is literally missing `revert_task_tool`.
- **~5GB of the 5.3GB in `worktrees/`** is stale/dormant because GC never sweeps worktrees for cancelled/stalled/failed tasks.
- Debug/perf-check layer (watchdog, GC, 27 test files, cost-guardian) is **already proportionate, mostly silent** — not the bloat source. It under-reaches; it doesn't over-reach.
- Biggest **token** lever is ~150-200+ irrelevant `ecc:*` domain skill packs enumerated into every session despite being CLAUDE.md-flagged "hard skip" — CLAUDE.md can stop Claude *invoking* them, not the harness *listing* them.
- Real duplicate-file clusters exist (4 near-identical FB poster scripts, 6 near-identical scene-gen pipelines, 2 dead one-campaign clusters).

## 1. Strengths — keep

| Area | Evidence |
|---|---|
| Terminal-status race guard | `lib/db.py:326-389` — folds the check into the UPDATE's WHERE clause, not read-then-write. Has a regression test. |
| merge_task no-op protection | `tools/git_ops.py:178-283` — 4 independent guards, each comment citing the incident it closed (#13/#29/#42). |
| recall()/reflect() | Deliberately reject embeddings ("keyword ranking over SQLite is enough at this scale"). Read-only. |
| TOON encoder | `lib/toon.py`, ADR-0010 — real, shipped, measured token savings. |
| Watchdog pid-gated tab closure | `runners/watchdog.py:157-170` — won't auto-kill a manually-attached human session. |
| Worktree cleanup trigger point | `tools/git_ops.py:336` — correctly placed at the one guaranteed-safe moment. |
| Wiki split (ADR-0013) | org 993 + mooniex 1,524 lines, zero overlapping section numbers. |
| ADR discipline | 16 + 30 decision files with dates/attribution. |
| Org-tooling test coverage | 27 `test_*.py` — spawn/tab/owner_cto routing, iTerm typewriter. |

## 2. Weaknesses — as originally ranked

**Critical:** W1 three drifting orchestration-tool implementations · W2 owner_cto guard on only one of three paths · W3 `depends_on` invisible to the lock/collision layer
**High:** W4 retry-before-claim double-spawn window · W5 worktree GC gap (source of the 5.3GB) · W6 one worktree stuck at an unreachable status
**Medium:** W7 dead code · W8 schema drift (`claudesign_project_id`) · W9 load-bearing files untracked · W10 `.asset-cache/` dead weight · W11 fomo-scanner cron documented but not installed · W12 duplicate desk-claim hooks · W13 stale docs · W14 abandoned scratch dirs

## 3. Excess — cut or merge
`runners/dev.py` · `db.is_session_busy` · `claudesign_project_id` · `.asset-cache/` (72MB) · `testsprite/` + `demo-api/` · SpaceX cluster (4 files) · Frost Knight cluster (3 files) · 4× `post-*.js` · 6× scene-gen scripts · the three orchestration tool surfaces.

## 4. Debug / perf-check systems — verdict
**Proportionate, not the bloat source.** Almost every hook prints nothing or ≤3 lines. The real problem is under-reach: GC didn't clean worktrees (W5), nothing monitors whether fomo-scanner's cron is installed (W11).

## 5. Token footprint
| Source | Size | Auto-loaded? |
|---|---|---|
| 3× CLAUDE.md | 285 ln / 16KB | Yes — already lean |
| `memory/MEMORY.md` index | 113 ln / 20KB | Yes — by design |
| `memory/*.md` (113 files) | 3,443 ln / 324KB | No — on demand |
| IRON-RULES ×2 | 2,517 ln | No — wiki_read on demand |
| **Available-skills listing** | 104 local + 58 user + 229 ecc + 8 finance + 4 cost-guardian + ~6 meigen | **Yes, in full, every session** |
| 18 hook scripts | 1,834 ln | Mixed |

---

# STATUS — remediation complete 2026-08-07 (CTO)

CEO approved three waves. **All 14 weaknesses are now closed, fixed-by-decision, or corrected-as-wrong.** Everything below was verified by the CTO independently re-running tests and reading code, not by trusting DEV reports.

## ✅ Fixed and merged

| # | Fix | Evidence |
|---|---|---|
| W1+W2 | REPL tool parity + shared ownership guard (`lib/task_ownership.py`) | task-b5db982c, sha `0d7f916` |
| W1 (root cause) | Manifest consolidation — `lib/org_tools_registry.py` is now the single source of truth; all 3 surfaces generate from it; `scripts/test_tool_parity.py` fails if any surface drifts | task-78ef13b0 `d637c75` + task-724cff99 `73253d5`. Net **+298/−533 lines** — the codebase shrank. |
| W3 | `depends_on` enforced at delegate — `db.unmet_dependencies()`; refusal leaves status untouched (so it is distinguishable from a touches-collision, which sets `conflict`) and explains itself in `delegate_log` | task-e8086826, sha `0951537` |
| W4 | Double-spawn closed at source — grace-period check on `updated_at` for unclaimed spawns, plus a live-pid check before any re-delegate reset (won't orphan a running DEV) | same |
| W5 | Worktree reap on terminal status + `--reap`/`--go` CLI | task-2d7bc187, sha `cdeffb3` |
| W6 | `merged` added to `TERMINAL_STATUSES` after tracing that **no code writes it** (legacy value; the one live row was hand-set in the DB after a GitHub-PR merge) | task-c1604acf, sha `affa3f0` |
| W7 | `runners/dev.py` + `db.is_session_busy` deleted | `cdeffb3` |
| W8 | `claudesign_project_id` dropped — idempotent guarded migration. Ran against the live 379-row DB: column gone, **379 rows intact, `integrity_check: ok`**. Backup at `~/.config/mooniex/db-backups/`. | `0951537` |
| W9 | Blanket `.claude/` ignore replaced with precise rules; 2 skills + 2 hooks + the repo-local `.claude/settings.json` now tracked; `scripts/test_loadbearing_tracked.py` fails if a load-bearing file goes untracked again | task-d3df329f, sha `07f8c3c` |
| W10/W14 | `.asset-cache/` (72MB), `testsprite/`, `demo-api/` → Trash (reversible) | 2026-08-06 |
| W13 | Corrected `config/wikis.yaml`'s false "org wiki doesn't exist on disk" comment + 2 hook files citing `LLMs/IRON-RULES.md §1/§17` (those sections live in `org:` post-ADR-0013) | sha `1bf56d9` + `~/.claude/hooks/office-{inbox,pretool}.mjs` |
| Token | 76 irrelevant `ecc:*` skills set `"off"` via `skillOverrides` in `~/.claude/settings.json` | 2026-08-06 |
| Disk | **5.3GB → 418MB.** 13 stale `review` worktrees removed with CEO sign-off after confirming `remove_worktree` defaults `delete_branch=False` — **all 13 branches survived, zero work lost**; recovery table + branch SHAs parked in LungNote. | 2026-08-07 |

## ⚠️ Findings this document got WRONG (corrected when acted on)

1. **W9's "91 symlinks into `.agents/`" is false.** `.claude/skills/` holds **13 real dirs + 1 symlink**. The 91 skills in `.agents/skills/` are a third-party wealth-management pack (account-maintenance, KYC, AML…) that nothing under `.claude/skills/` points at. The real exposure was far smaller: 2 org skills + 2 hooks.
2. **W9 also understated the problem in one place.** The hooks are wired from the **repo-local** `.claude/settings.json`, not `~/.claude/settings.json` as the audit implied. That file was itself untracked — so tracking only the scripts would have produced a clone where the hooks exist but never fire. Caught by the DEV, not by the audit.
3. **W12's premise is false.** The two office hooks do **not** fire two redundant API calls per turn — they **both no-op every turn**. Last successful desk claim: **2026-05-18, 81 days ago**; `MOONIEX_DESK` unset; no office cache for any current cwd. They gate on the Desk pattern, retired 2026-05-28 by ADR-0007, and `mcp__mooniex-coord__*` supersedes them. Merging two dead hooks would have been worthless work.

## 🟡 Open — CEO decisions, not code work

- **W11** — `com.mooniex.fomo-scanner.plist` is still not installed; the docs claim an every-4h scan that has never run. Decision: install it (a recurring background job that may consume API budget) or delete the files so docs match reality. Not actioned unilaterally.
- **W12** — given the correction above: remove the two dead office hooks from `~/.claude/settings.json`, or revive the office system. Not actioned — it is the CEO's global config and touches cross-session coordination.

## 📋 Still open, deliberately not scheduled
Excess consolidations (SpaceX/Frost-Knight campaign clusters, `post-*.js` ×4, scene-gen ×6) and 5 of the 9 original rule recommendations (campaign-script retention rule, skill-authoring guidance, merge-checklist extension, and writing the "one tool-surface per role" IRON rule — the *code* now enforces it, the *rule* isn't written). Low urgency; none is load-bearing.

## Lesson recorded
Three of fourteen findings from a 4-agent parallel audit were wrong in ways that only surfaced when a human-directed agent acted on them and verified against the real system. **Audit findings are hypotheses, not facts** — re-verify each one at the moment of acting, and give the DEV the corrected numbers rather than a wiki link. Every fix in the table above was independently re-tested by the CTO after merge, which is how the migration integrity, the branch survival, and the parity results are known rather than assumed.
