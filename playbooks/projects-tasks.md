# Playbook — Project + Task tracking (mooniex-webapp)

> Owner: CTO. Last refresh 2026-04-29.
> Mandatory rule for **every Dev / agent**: see IRON-RULES §19.

## Why this exists

MCP message bus (`webapp_office_messages`) handles **conversation**.
This playbook covers the **work-state layer** — what's todo, who
owns it, what blocks what, what's done. CEO directive 2026-04-29:
draft M9-M12 milestone work into Project + Tasks first so plans
don't drift from execution.

Not duplicate of MCP. Complement. Rule of thumb:
- Need to ask "is anyone working on X?" → Tasks.
- Need to ask "what did the code reviewer say about X?" → MCP
  thread linked to that task.

## Surfaces

| URL | Use |
|---|---|
| `/admin/projects` | List of milestone projects + progress bars |
| `/admin/projects/[slug]` | Kanban (5 cols) + List view + create task |
| `/admin/tasks/[id]` | Full task edit (status / priority / owner / depends_on / hours) |

API is admin-gated. Anyone with the admin role flag can read +
write. RLS on the underlying tables enforces this.

## Lifecycle (every task you pick)

```
                       ┌─────────┐
                       │  todo   │ ◀────────── new task created
                       └────┬────┘
                            │  click Claim (atomic)
                            ▼
                  ┌──────────────────┐
                  │   in_progress    │ ◀─── you are working on it
                  └─────┬─────────┬──┘
                        │         │
                        │         │ blocked by external dep
                        │         ▼
                        │   ┌─────────┐
                        │   │ blocked │
                        │   └────┬────┘
                        │        │
                        ▼        │ unblock
                   ┌─────────┐   │
                   │ review  │ ◀─┘
                   └────┬────┘
                        │ admin / peer signs off
                        ▼
                   ┌─────────┐
                   │  done   │  completed_at + completed_by_email auto-stamped
                   └─────────┘
```

Drop / give up:
- **Release** → status flips back to `todo`, `owner_desk` cleared.
  Anyone else can claim it.
- **Cancel** → admin only, status `cancelled`. Use for tasks no
  longer relevant (scope cut, duplicate).

## Pick → Work → Close

### 1. Set "I am" once

`/admin/projects/[slug]` toolbar has an `I am: <picker>` dropdown.
Set it to your desk (`Desk-A` … `Desk-CTO`). Stored in localStorage
— survives navigation.

### 2. Filter to "Available to me"

Click the `Available to me` pill. Shows only:
- `owner_desk IS NULL` (anyone can claim)
- OR `owner_desk = <my desk>` (already mine)

Excludes done / cancelled / owned-by-other.

### 3. Claim a task

Click `Claim` on a card. Atomic — race-safe even if two desks
click at the same second. Effects:
- `owner_desk` ← your desk
- `owner_agent_id` ← (optional, if MCP agent_id is known)
- `status` ← `in_progress`
- `updated_by_email` ← your admin email

If 409 returned: someone else got it first. Pick another card.

### 4. Work on it

Outside of this UI:
- Branch off `bootstrap/landing-mvp`
- Code, commit, push (pull-before-edit per IRON §1)
- Reference task id in commit footer: `Task: <uuid>`
- Use MCP messaging for questions / blockers / reviews

### 5. Close

Three terminal states:
- **done** — work shipped. Click status dropdown → `done`. The
  trigger auto-stamps `completed_at`. `updated_by_email`
  becomes the sign-off record.
- **review** (intermediate) — PR up, waiting for peer / CTO. Move
  to `done` after merge.
- **blocked** — waiting on dep. Set `depends_on` on this task to
  point at the blocker uuid. Move back to `in_progress` when
  unblocked.
- **cancelled** — drop entirely (admin-only).

### 6. Or release

`Release` button → owner cleared, status back to `todo`. Use when:
- You realised it's not your scope
- You're done for the day and don't want to hold the lock
- Another desk explicitly asked to take over

Release does NOT cancel the task — just hands it back to the pool.

## Picking work that won't collide with others

Three guards prevent overlap:

1. **owner_desk lock** — only one desk can hold an `in_progress`
   task at a time. Atomic claim on POST /api/admin/tasks/[id]/claim
   — second claim returns 409.
2. **depends_on graph** — if task B depends_on [A], don't claim B
   while A is still in `todo` / `in_progress`. The kanban shows
   a dep badge on the card.
3. **Project boundary** — projects are separate streams. Claiming
   a task in `m9-compliance` doesn't conflict with `m11-trading-core`
   work. Pick a project that isn't being worked by another desk.

If two devs *must* work the same task (pair coding etc.), use
sub-tasks: claim the parent, create child tasks as `parent_task_id`,
each child has its own owner.

## Rules of engagement (mandatory)

1. **Every actionable Dev work item ≥ 30 min must have a task
   row** *before* the code work starts. No "shadow tasks" tracked
   in personal todo files / chat.
2. **Claim before code.** Don't open files for a task you haven't
   claimed. Two desks editing the same scope = merge hell.
3. **Status transitions go through this UI** (or PATCH route).
   Don't UPDATE the table directly via Supabase SQL editor —
   skips audit trail (`updated_by_email`).
4. **Leave a finish trail.** Set `actual_hours` on close. Sign-off
   via `updated_by_email` happens automatically.
5. **Don't claim more than 3 in_progress tasks at a time.** Kanban
   work-in-progress limit. Finish or release before grabbing more.
6. **If blocked > 24 h, raise it.** Flip status to `blocked`,
   set `depends_on`, DM the dep owner via MCP. Don't sit silent.
7. **Sub-task to split.** Big task = parent + children. Children
   get their own claim cycle.
8. **Cross-cutting tasks need a label.** Tag `legal` / `pro-perk`
   / `pdpa` / `seo` so the dashboard filter works.

## CEO / admin actions

- **Create project**: list page → New project. Pick slug
  (lowercase + hyphens) + title. Optional: priority (lower =
  sooner), labels, doc_path link.
- **Re-prioritise**: edit project priority field. List re-orders.
- **Bulk seed**: `scripts/seed-projects-m9-m12.mjs` — reads plan
  file, creates rows idempotently (re-run skips dups via
  `metadata.seed_key`). Use as template for future
  `seed-projects-<milestone>.mjs` scripts.
- **Cancel a task**: status → `cancelled`. Doesn't delete row.
- **Delete a task**: trash button. Hard delete. No undo.

## Verification

1. Login admin → `/admin/projects` shows 4 milestone projects
   (M9 / M10 / M11 / M12) seeded by the script.
2. Click `m9-compliance` → kanban view 11 todo tasks.
3. Pick `Desk-A` in "I am" dropdown → toggle "Available to me"
   filter → all 11 visible (no owners yet).
4. Click `Claim` on first card → status flips to `in_progress`,
   border green, owner_desk = `Desk-A`.
5. Open `/admin/projects/m9-compliance` in another browser tab as
   Desk-B → that card shows "Owned by Desk-A", no Claim button.
6. Click `Release` (Desk-A tab) → back to todo, no owner.
7. Status dropdown → `done` → `completed_at` populated.
8. `/admin/tasks/[id]` page shows full edit form including
   `depends_on` checkbox picker against sibling tasks.

## When to bypass (rare)

- Hot incident < 30 min, code change ships before anyone else
  sees the screen → write a retrospective task afterwards.
- One-line typo fix on docs / CLAUDE.md / wiki — overhead > value.
- Anything else: there's a task or there's a problem.

## See also

- `IRON-RULES.md` §19 (canonical)
- `playbooks/cron-rules.md` — cron + Pro tier matrix
- `playbooks/fal-queue-jobs.md` — async toolkit
- `src/lib/projects/` — types + queries (server-only)
