# ADR — Web Designer autonomous spawn on the iterm backend

- **Date:** 2026-06-15
- **Status:** Accepted
- **Owner:** CTO
- **Commit:** `d92cb09` (`runners/dev_init.py`)
- **Proof:** task-a29b1fec built `design-templates/admin-finance-dashboard/example.html` + `TEMPLATE.md` autonomously (commit `4f238eb3`, Chrome-verified)

## Context

`runners/dev_init.py` short-circuited **every** `web_designer` spawn to a
passive `tail -F /tmp/mooniex-mirror-<task>.log` viewer — a leftover from
the era when claudesign used `spawn_backend: tmux` and the open-design Web
UI drove a bridge (`tools/claudesign_tmux_bin.py`) that spawned a real
claude per chat message.

claudesign switched to the default **iterm** backend on 2026-05-23 (the
intent recorded in `projects.yaml`: *"web_designer spawns interactive claude
TUI in tab … so CTO can use send_to_dev like other DEV roles"*), but
`dev_init` was never updated to match. Result: a CTO `delegate_task` on a
`web_designer` task **never came up live** — 0-byte mirror, no `claude`
process, the tab just printed "Type prompts in the claudesign Web UI."
Both the CFO and the CTO hit this independently and read it as "spawn dies."

## Decision

Gate the passive viewer on `spawn_backend == "tmux"` only. On the iterm
backend (the current claudesign config), `web_designer` **falls through to
the same autonomous `claude` TUI exec as `developer`**, plus:

1. the web_designer role doc (`roles/web_designer.md`) as `--append-system-prompt`;
2. the **`Skill` tool** added to `--allowed-tools` (so it can use the org
   UI-craft skills: frontend-design / mooniex-tool-builder);
3. the project's design source resolved from the **claudesign project UUID**
   in the task description and appended to the prompt
   (`lib/db.designer_kickoff_suffix`), so the agent knows which `.od`
   brand/theme to match without the gitignored `.od/` in its worktree.

## Consequences

- **CTO can now `delegate_task` a `web_designer` exactly like a `developer`**
  — autonomous, in an iTerm2 tab, drivable with `send_to_dev`. The only
  extra required input is the project UUID in the task description
  (playbook `web-designer.md` §9).
- The **CEO-driven Web UI flow is unchanged and separate**:
  `scripts/spawn-web-designer.sh` (now also the `/spawn-web-designer` skill)
  boots the open-design app on `localhost:3000` for hands-on design.
- The legacy tmux+bridge viewer still works if `spawn_backend` is ever set
  back to `tmux` (documented as playbook §4c).
- `tools/cto_chat_designer.py` (a viewer-era workaround that spawned its own
  `claude --continue`) is now redundant; `send_to_dev` suffices.
- **Verification rule:** after any web_designer spawn, confirm the process
  is a `claude` TUI (not a `tail`), not just `status=in_progress`
  (see memory `feedback_dev_first_delegate_silent_death`).

## Related

- Playbook: `playbooks/web-designer.md` §4 (three spawn modes), §9 (spawn inputs)
- Memory: `web-designer-harness-vs-developer`, `designer-spawn-inputs`
- Prior ADR: `decisions/ADR-001-claudesign-fork-strategy.md`
