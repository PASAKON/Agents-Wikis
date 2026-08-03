# CTO ↔ DEV orchestration conventions

Source of truth for how the CTO spawns developers, talks to them, knows
when they finish, persists blockers, and cleans up tabs.

**Last updated:** 2026-05-18 by CEO directive (rate-limit recovery
section added same day).

---

## 1. Spawn

- CTO calls `mcp__org__delegate_task` (or `delegate_parallel_tasks`,
  max 3 at once). This opens a new iTerm tab and runs
  `python -m runners.dev_init <role> <task_id>` which then `exec`s
  Claude Code TUI scoped to the DEV's worktree.
- Tab title is `<RoleDisplay> (<full task_id>)`. Do **not** truncate to
  6 chars — collisions on first hex digit will misroute messages.
  (`tools/delegate.py:39`)
- Permission mode is **`auto`** for both CTO (`scripts/cto-claude.sh:18`)
  and every DEV (`runners/dev_init.py:103`). Never spawn a DEV with
  `default` or `acceptEdits` — auto is the project default.

## 2. CTO → DEV chat

- Use `python -m tools.send_to_dev <task_id> "<message>"` to type a
  message **directly into the DEV's iTerm tab** with the `[CTO]:`
  prefix.
- This is the only sanctioned way for the CTO to instruct a running
  DEV mid-flow. **Do not** issue side-channel MCP calls — work must be
  visible in the DEV tab so the CEO can watch the keystrokes land and
  the claude TUI respond in real time.
- Background MCP delegation that blocks (`delegate_task`'s
  `_wait_for_terminal` poll) is fine for initial spawn but should not
  be used as a chat substitute.

## 3. DEV → CTO reply relay

Every DEV worktree gets a `.claude/settings.local.json` (auto-written
by `runners/dev_init.py:_write_dev_settings`) that wires a Stop hook
`scripts/hook-log-dev-reply.py`. The hook:

1. Reads the DEV's last assistant text from `transcript_path`.
2. Appends `[<ts>] Dev:<task_id>: <text>` to `state/logs/cto.log`.
3. Persists `session_id` + `last_checkpoint` back to `tasks` (see §8).
4. Detects rate-limit signals and flips status to `rate_limited` so
   the auto-resume scheduler can recover the work (see §8).
5. CTO's `UserPromptSubmit` hook tails cto.log and injects new lines
   into the CTO's next turn — so DEV replies surface in the CTO chat
   automatically.

Gating: hook self-checks `DEV_TASK_ID` env. No-op if unset.

In addition, DEVs can still call:

- `mcp__org__dev_message` — explicit one-line progress checkpoint.
- `mcp__org__submit_report` — final hand-off (sets status `review`).

## 4. Tab lifecycle

| Task status | Tab action |
|-------------|------------|
| `pending` / `in_progress` | tab open, DEV working |
| `review` | tab open, DEV idle waiting for CTO merge/reopen |
| `done` (post-merge) | **auto-close** via `tools/itermtab.py:close_tab` invoked from `git_ops.merge_task` |
| `rate_limited` | tab usually exited; auto-resume scheduler will re-open in a new tab when `retry_after_ts` passes (see §8) |
| `failed` / `cancelled` | tab open until CEO inspects |
| blocked waiting for CTO answer | tab open, DEV idle |

The rule: a tab closes when its task is **complete** (merged). Until
then the tab stays open so the DEV can resume if asked.

If a tab is closed manually mid-flow, the underlying claude process
keeps running unless killed. To truly cancel, kill the PID then reset
the task status to `pending`.

## 5. Blocker → GitHub issue (always)

Any time a DEV cannot proceed — missing secret, ambiguous spec, broken
dependency, external service down — the DEV **MUST** file a GitHub
issue on the project repo before reporting the blocker. This is a hard
rule because tabs can be closed, the user's machine can sleep, or the
network can drop; the issue is the durable record that lets work
resume.

Mechanism:

- DEV calls `mcp__org__file_blocker_issue(title, body)`.
- The tool runs `gh issue create` on the project's GitHub remote with
  body prefixed by the task id and worktree path so a future agent can
  reattach.
- Returned issue URL is logged via `dev_message` so the CTO sees it.

CTO MUST verify the issue link exists before marking a task `failed`.
If a DEV reports a blocker without an issue, CTO reopens with feedback
"file GitHub issue first".

## 6. CTO learns task is done — checklist

In priority order:

1. UserPromptSubmit hook auto-injects `Dev:<task>: <reply>` into next
   CTO turn (live relay, see §3).
2. `mcp__org__submit_report` flips status to `review` → cto.log entry.
3. Manual poll: `sqlite3 state/tasks.db "SELECT id,status FROM tasks
   WHERE status IN ('review','done','failed','rate_limited')"`.

CTO never assumes a tab being open means work is in progress — always
check status in DB.

## 7. Cross-references

- `runners/dev_init.py` — spawn DEV claude TUI + write settings.local
- `runners/dev_resume.py` — soft-resume launcher (uses `claude --resume`)
- `runners/auto_resume.py` — scheduler poller for `rate_limited` tasks
- `tools/delegate.py` — open iTerm tab, poll for terminal status
- `tools/send_to_dev.py` — visible CTO → DEV chat
- `tools/resume_dev.py` — open resume tab (soft or hard)
- `tools/itermtab.py` — close tab on merge
- `tools/gh_issue.py` — GitHub issue helper
- `scripts/hook-log-dev-reply.py` — DEV reply relay + RL detection
- `scripts/hook-log-reply.py` — CTO reply log (gated `CTO_SESSION=1`)
- `runners/dev_mcp_server.py` — DEV-side MCP tools
- `runners/cto_mcp_server.py` — CTO-side MCP tools

## 8. Rate-limit recovery

When Anthropic returns a 429 (or any usage-limit / quota error) the
claude TUI in the DEV tab stops and shows an error. The orchestration
layer treats this as a recoverable pause, not a failure:

### Detection
`scripts/hook-log-dev-reply.py` scans the transcript on every Stop
event for the regex
`(rate[_-]?limit|rate limited|429|too many requests|usage limit|quota exceeded)`.
If matched it:
- sets `tasks.status = 'rate_limited'`
- writes `tasks.retry_after_ts = now + 5min` (default backoff)
- logs `[ts] RateLimit Dev:<task>: ...` to cto.log

### State preservation
Every Stop fire also writes:
- `tasks.session_id` ← claude session id (transcript filename stem)
- `tasks.last_checkpoint` ← ISO 8601 UTC timestamp

The worktree branch keeps every commit + uncommitted diff. TASK.md +
role doc + system prompt are reproducible. **Nothing is lost**.

### Resume — soft (preferred)
`python -m tools.resume_dev <task_id>` opens a fresh iTerm tab that
execs `claude --resume <session_id>` via `runners/dev_resume.py`.
Chat history, scratchpad, in-progress thinking — all preserved. The
launcher prepends a `[RESUMED]` nudge that tells the DEV to run
`git status` + `git diff` before doing anything else.

### Resume — hard (fallback)
If `session_id` is missing (e.g. transcript rotated, hook never fired),
`dev_resume.py` falls back to `runners/dev_init.py` which spawns a
fresh session. The DEV reads TASK.md + git state to figure out where
it left off — work survives even if chat history is gone.

### Auto-resume
`python -m runners.auto_resume` polls every 60s for tasks where
`status='rate_limited' AND retry_after_ts <= now()`, then calls
`resume()` for each. Logs through `lib.notify.info` so every resume
shows up in the CTO chat via UserPromptSubmit. Spawn this in its own
iTerm tab once per CTO session (or wire into `scripts/spawn-cto.sh`).

### Backoff policy
Default is a flat 5-minute cooldown per RL hit. Exponential backoff
(5m → 15m → 60m on consecutive hits) is a future enhancement — for now
if a task RLs repeatedly the CEO can either kill it manually or let
the scheduler keep retrying.

### CEO escape hatch
- Force resume now: `python -m tools.resume_dev <task_id>`
- Inspect what's stuck: `sqlite3 state/tasks.db "SELECT id, status,
  retry_after_ts FROM tasks WHERE status='rate_limited'"`
- Reset to pending (full restart, loses session continuity):
  `UPDATE tasks SET status='pending', session_id=NULL,
  retry_after_ts=NULL WHERE id='task-XXX'`
