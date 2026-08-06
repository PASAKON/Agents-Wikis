# mooniex-agents (orchestration meta-repo)

Repo: `/Users/gob/Projects/Agents` · GitHub `PASAKON/mooniex-agents` · default branch `main` (direct push allowed, CTO merges).

The virtual-org runtime: CEO → CTO → DEV orchestration, task queue (`state/tasks.db`), iTerm visible-chat plumbing, MCP servers, watchdogs.

## Changelog

### 2026-06-12 — DEV-report routing hardened (issue #15) — CTO 37ad539a
- **Problem:** tasks created by non-CTO CXO sessions (CFO etc.) had `owner_cto=NULL` → DEV reports broadcast into EVERY open CTO tab (`send_to_cto` legacy path). Wrong CTO nearly merged foreign work; stay-in-lane rule was the only guard.
- **Fix (merge `4beb8d9`, task-2094c430):**
  - `lib/db.py` stamps `owner_cto` from `CTO_SESSION_ID` → `CXO_SESSION_ID` fallback; new additive `owner_role` column; loud stderr warning on ownerless create.
  - `tools/send_to_cto.py`: ownerless messages now orphan to `state/orphan-dev-replies-unowned.log` — **no broadcast** unless `SEND_TO_CTO_BROADCAST=1`. Winid lookup is role-aware (`<role>-<id>.winid`), so CFO-spawned DEV reports route to the CFO tab.
  - `DEV_CTO_ROLE` threaded through `dev_init` → `dev_mcp_server` / `hook-log-dev-reply`.
  - `scripts/backfill_owner_cto.py` — manual stamping of in-flight ownerless rows (`--list`, `--task --owner [--role]`). No auto-attribution by design.
- **Operational notes:** running C-level sessions must restart to pick up the change; lane owners backfill their own in-flight tasks (e.g. CFO lane task-1112d9d7).

### 2026-06-12 — paste-stall fixed (merge `0c4188b`, task-0d5f21e2) — CTO 37ad539a
- **Problem:** multi-line messages typed into claude TUI tabs stuck in the composer unsubmitted — body + CR written back-to-back, CR swallowed by bracketed paste (CEO screenshot 2026-06-12, recurring).
- **Fix:** shared `lib/iterm_type.type_submit_fragment()` — body → `delay 0.4` → CR → `delay 0.3` → rescue CR (no-op if first submitted). All 9 typewriter sites routed through it: send_to_cto/send_to_dev/send_to_cxo/inject_prompt via helper, idle-ping-watcher.sh + cxo-claude.sh inlined. New suite `scripts/test_iterm_typewriter.py` (8 tests). 47/47 green.
- **Operational note:** running watchers/sessions use the old code until restarted.

### 2026-08-06 — Full architecture audit (CEO-requested) — CTO
- **Ask:** find weaknesses to fix, strengths to keep, excess to cut/merge (incl. debug/perf-check tooling), token-reduction levers, skill/rule additions.
- **Method:** 4 parallel read-only Explore agents, ground-truth verified (not memory-derived) — core engine, scripts/tools/roles, disk-bloat root-cause, wiki+skills+hooks token footprint.
- **Full findings + evidence:** `org:reference/2026-08-06-agents-system-audit.md`.

### 2026-08-06 — Audit remediation wave 1: worktree reap + critical orchestration-gap fixes — CTO
- **task-2d7bc187** (merge `cdeffb3`): `tools/gc_stale_tasks.py` reclaims worktrees on terminal status + `--reap`/`--go` CLI for the backlog. Deleted `runners/dev.py`, `lib/db.py`'s `is_session_busy`.
- **task-b5db982c** (merge `0d7f916`): closed W1 (REPL missing 4 tools) and W2 (ownership guard only in `cto_mcp_server.py`) — extracted `lib/task_ownership.py`.
- **Also applied**: `~/.claude/settings.json` `skillOverrides` off for 76 irrelevant `ecc:*` skills; `.asset-cache/`/`testsprite/`/`demo-api/` → Trash.

### 2026-08-06 — Audit remediation wave 2: manifest consolidation (recommendation #1) — CTO, DONE
CEO approved the audit's biggest structural recommendation with explicit instruction to verify thoroughly. Split into 2 serialized steps.
- **Step 1/2** (task-78ef13b0, merge `d637c75`): built `lib/org_tools_registry.py` (17 `ToolSpec`s + `dispatch()`) as a pure addition. `scripts/test_org_tools_registry.py` proved byte-equivalence against `cto_mcp_server.py`'s real output (28/28, CTO-reran independently). Standardized TOON formatting + one uniform error wrapper (9/17 tools previously had none). Investigated CEO's flagged `owner_cto` concern — not a live bug (`lib/db.py` already falls back to env), registry stamps explicitly anyway. Found+fixed `cto.py`'s `merge_task` missing `override_touches_check`.
- **Step 2/2** (task-724cff99, merge `73253d5`): wired all 3 production files to the registry.
  - `cto_mcp_server.py`: **thin-stub fallback** (17 one-liners calling `registry.dispatch_sync`/`dispatch`) — investigated FastMCP's actual source first (`.venv/.../mcp/server/fastmcp/`); confirmed `Tool.from_function` always does `inspect.signature()`, no explicit-schema registration API exists, so full dynamic codegen wasn't safely possible. Correct, evidence-based call, not a shortcut.
  - `cto.py` / `cto_chat.py`: **fully dynamic** — zero hand-written per-tool bodies left, both derive their tool list from `registry.REGISTRY` at import time.
  - New `scripts/test_tool_parity.py`: asserts registry names == `cto_mcp_server.py`'s live FastMCP-introspected names == `cto.py`'s == `cto_chat.py`'s. DEV proved it has real teeth (scratch-added a fake 18th tool at runtime, confirmed the test flips to FAIL, reverted — nothing committed). This is the actual fix for W1 — structurally can't silently drift again.
  - Registry gained `dispatch_sync()` (FastMCP's sync stubs run inside FastMCP's own event loop; `asyncio.run()` there would raise) — one extra file outside originally-declared touches, reviewed via `review_diff` + independently re-tested before `override_touches_check=True` merge.
  - Found (unprompted) two more drift instances of the SAME W1 bug class: `main.py`'s one-shot path was also only wiring 14/17 tools (recall/reflect/revert_task_tool were dead code there too); `cto.py`/`cto_chat.py` used the wrong canonical name `delegate_parallel` vs the rest of the org's `delegate_parallel_tasks` — both now fixed.
  - Flagged (not silently changed): `cto.py`'s SDK responses now set `isError` consistently (derived from `"ERROR:"` prefix) instead of the old ad-hoc scheme, which had mismarked `review_diff`'s "no worktree" / `reopen_task`'s "not found" as errors. CTO reviewed and approved — net correctness improvement, low blast radius.
  - **Net result**: 6 files, +298/−533 lines — the codebase got *smaller* while duplication went to zero.
  - CTO independently reran the full test suite + FastMCP live introspection directly on `main` post-merge (not just trusted the DEV report): 17/17 tools registered correctly, only 2 pre-existing unrelated failures (`test_tm_prompt_loop.py`; `test_trader_mindset_batch.py` — debug-mantra'd this one live, confirmed via isolating `data/trader-mindset-topics.json` that it fails identically clean or dirty, i.e. a stale "seed pool ships all-unused" assumption violated by real production posting history, zero connection to this merge).
- **Deferred, not done**: none — both steps of the recommendation are complete. `depends_on` lock-layer enforcement (W3) and the remaining Medium-severity audit findings are still open, unscheduled.
