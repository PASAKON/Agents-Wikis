# Playbook — Before editing a folder that's not your home folder

Run through this **every time** you're about to touch a file outside your current project.

## Step 1 — Identify edit rights

Check [INDEX.md](https://github.com/PASAKON/MoonieX-Wikis/blob/main/INDEX.md) (`mooniex:INDEX.md`) "Edit rights" column for the target folder.

| Edit rights | What to do |
|-------------|------------|
| **Full edit** | Continue to Step 2 |
| **Read-only** | **STOP**. Ask the user explicitly: "I need to modify `<folder>` to do X — OK?" Wait for confirmation. Don't write until then. |

Currently full-edit folders: any `mooniex-webapp (Desk-*)` desk you've claimed via the Office Simulator (`/api/office/claim` returned 200), and `mooniex-option`. Everything else = ask first.

## Step 2 — Read the project's own rules

Open [`projects/<target-folder>.md`](../projects/) and skim:
- Iron rules section
- Cross-folder relationships
- Recent incidents

Then open the **source** files referenced (`CLAUDE.md`, `AGENTS.md`, `setup-windows.md`) — wiki may be stale.

## Step 3 — Sync git state

```bash
cd "<target-folder>"
git fetch origin
git status
git log --oneline HEAD..origin/<branch>      # commits behind — must be 0 before editing
```

If behind: `git pull --ff-only` (or `--rebase`).

If you're not on the right branch (per project's branch convention, see [IRON-RULES §2](https://github.com/PASAKON/MoonieX-Wikis/blob/main/IRON-RULES.md#section-2--github-push-rules-cross-cutting) (`mooniex:IRON-RULES.md`)): create one.

## Step 4 — Check OTHER desks (shared repo)

For every other webapp desk folder:

```bash
for d in Desk-A Desk-B Desk-C; do
  echo "=== $d ==="
  git -C "/Users/gob/projects/mooniex-webapp ($d)" status -sb 2>/dev/null
done
```

Or query the Office Simulator: `curl -s https://www.mooniex.com/api/office/state | jq '.active'`

If the other folder has uncommitted work or unpushed commits → coordinate (don't deploy until resolved).

## Step 5 — Verify env / database conventions

- **DB tables**: only touch tables with the prefix this project owns ([IRON-RULES §4](https://github.com/PASAKON/MoonieX-Wikis/blob/main/IRON-RULES.md#section-4--database--supabase) (`mooniex:IRON-RULES.md`))
- **Env vars**: if you need to add one, use `printf` not `echo` for Vercel; for `mooniex-option` use `env-push.js`
- **Migrations**: `YYYYMMDDHHMM__<descriptor>.sql`, add domain prefix on collision

## Step 6 — Do the work

Edit. Test. Commit on the right branch.

## Step 7 — After commit

- **Push immediately** (don't sit on local commits — incident 2026-04-24)
- For `mooniex-claudeflow`: only Claude pushes, Codex stops at `git commit`
- If deploying: switch to [vercel-deploy-coordination.md](https://github.com/PASAKON/MoonieX-Wikis/blob/main/playbooks/vercel-deploy-coordination.md) (`mooniex:playbooks/vercel-deploy-coordination.md`)

## Common pitfalls

- **Editing `/Users/gob/projects/mooniex-webapp/`** when you should edit MXN — both have `CLAUDE.md` saying "edit MXN". Read the file first.
- **Forgetting that MXN and upstream share a Vercel project** — deploying from MXN affects www.mooniex.com immediately.
- **Editing `mooniex-option/.env`** without using `env-push.js` — secret won't sync to other machines.
- **Adding a Supabase table without prefix** — breaks the prefix invariant. Use `webapp_*`, `option_*`, etc.
