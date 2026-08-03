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
