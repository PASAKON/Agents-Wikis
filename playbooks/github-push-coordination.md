# Playbook — GitHub push coordination

Mooniex has multiple repos with different push policies. Get this wrong → incidents like 2026-04-24.

## Branch convention by project

| Project | Branch pattern | Who pushes |
|---------|----------------|------------|
| `mooniex-webapp` + MXN | direct `main` (after pull-before-edit) | Both agents |
| `mooniex-claudeflow` | `claude/<feature>` (Claude) / `codex/<feature>` (Codex) | **Claude only.** Codex commits, Claude reviews + merges + pushes. |
| `mooniex-option` | direct `main` | User or Claude (from Windows machine) |
| `mooniex-controller-system` | n/a (different remote) | Per `geezerrrr/agent-town` convention |
| `mooniex-wa-system` | direct `main` | n/a (local-first) |

## The four golden rules (verbatim from `mooniex-webapp/AGENTS.md`)

> 1. **Pull before edit, every time.** `git fetch origin && git pull --ff-only` (or `--rebase`) before touching any shared file.
> 2. **Push immediately after commit.** Never leave local commits sitting.
> 3. **Migration timestamps must not collide.** `YYYYMMDDHHMM__<descriptor>.sql`. On collision, add domain prefix.
> 4. **Pull before `vercel deploy`.** Run `git rev-list --count HEAD..origin/<branch>` first.

## Conflict resolution

From `mooniex-webapp/AGENTS.md` verbatim:

> - **Same file, both edited, the other agent's change is already on prod:** take their version, re-apply your intent as a *follow-up* commit. Don't undo deployed work.
> - **Both in-flight (neither shipped):** resolve manually, run typecheck + build locally, then push.
> - **Shell / layout / TopBar:** never `git checkout --theirs` blindly. Merge props by hand — both agents add headers, sidebar items, etc.

## Conflict resolution preference per file class (from auto-memory)

Per `feedback_mooniex_multi_agent_coordination.md`:
- **Migrations**: never overwrite — always add a follow-up file
- **Shell / layout / TopBar**: hand-merge props
- **Generated files** (e.g. `next-env.d.ts`): take origin

## ClaudeFlow handoff workflow (verbatim from `mooniex-claudeflow/CLAUDE.md`)

For Codex tasks:

```
1. สร้าง task file: docs/ops/tasks/<number>-<name>.md (ใช้ template จาก docs/ops/TASK_TEMPLATE.md)
2. เปิด Codex แล้วสั่ง:

   อ่าน docs/ops/tasks/<number>-<name>.md แล้วทำตาม task ที่ระบุ
   กฎ:
   - อ่าน CLAUDE.md ก่อนเริ่มงาน
   - สร้าง branch ตาม Branch Convention ก่อน commit
   - แก้เฉพาะไฟล์ที่ระบุใน Lock section
   - ห้าม push

3. Codex ทำเสร็จ → แจ้ง user (branch + commit hash + ผลทดสอบ)
4. สั่ง Claude: "review branch codex/<name> แล้ว merge"
5. Claude: review diff → merge → push → sync VPS → bump version
```

## ClaudeFlow version bump (verbatim)

> - ทุกงานที่แก้โค้ด+ทดสอบผ่าน → `npm run -s version:test:bump`
> - deploy ได้เฉพาะเมื่อ user สั่ง → `npm run -s version:vps:promote`
> - รายงาน version ทุกครั้ง: `npm run -s version:report`

## Push checklist

```bash
git status                                    # working tree clean?
git log --oneline @{u}..                      # local-only commits visible?
git fetch origin
git log --oneline ..@{u}                      # behind? if so, pull first
git push origin <branch>
```

## When NOT to push

- `mooniex-claudeflow` from Codex (Codex never pushes — Claude is gatekeeper)
- Anything you didn't pull-before-edit (will conflict)
- Migrations with timestamps colliding with origin (rename first)
- Force-push to `main` (always blocked unless user explicitly asks)
