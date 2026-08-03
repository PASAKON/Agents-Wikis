# Playbook — Onboarding (first session as a new agent)

> When you arrive in a new ClaudeCode / Codex session and don't know where to start. Reads as the **single first thing** to load if you've never sat at a desk before.

> Owner: Secretary. Linked from [README.md](https://github.com/PASAKON/MoonieX-Wikis/blob/main/README.md) (`mooniex:README.md`) + [DEV-PROMPT-INLINE.md](../DEV-PROMPT-INLINE.md).

> Companion: [`local-dev-setup.md`](https://github.com/PASAKON/MoonieX-Wikis/blob/main/playbooks/local-dev-setup.md) (`mooniex:playbooks/local-dev-setup.md`) — once you can claim a desk, this gets you running the actual app.

## Pre-flight checks (≤ 30 seconds)

Run these once at session start. Each one answers one question; if the answer is wrong, fix before moving on.

```bash
# 1. Where am I?
basename "$PWD"
# Expect: "mooniex-webapp (Desk-X)" for Senior Dev / DevOps / ML / QA, or "LLMs" for Secretary, or a project folder name (mooniex-option, mooniex-claudeflow, etc).

# 2. What role does this folder map to?
# Open: /Users/gob/projects/LLMs/IRON-RULES.md §1 desk model.
# OR: cat ~/.claude/projects/-Users-gob-projects/memory/reference_office_role.md (if you have one)

# 3. Is the wiki up to date?
git -C /Users/gob/projects/LLMs status --short
# Expect: empty. SessionStart hook auto-pulls; if dirty, `git -C /Users/gob/projects/LLMs pull --ff-only` manually.

# 4. Is the office cache in place (folder-bound desks only)?
ls ~/.claude/office-state-*.json
# Expect: at least one. The PreToolUse hook claims a desk on first Edit/Write/Bash; cache appears then.

# 5. Any unread messages waiting for me?
# Folder-bound desks: the UserPromptSubmit hook auto-injects on the next prompt — no manual step.
# Secretary / Desktop user / curl: call mcp__mooniex-coord__read_inbox(for_agent=<your_id>, auto_ack=true).
```

## First-read order (when you've never opened this wiki)

Stop at the first one that answers your question:

1. **[README.md](https://github.com/PASAKON/MoonieX-Wikis/blob/main/README.md) (`mooniex:README.md`)** — the directory of every wiki file
2. **[IRON-RULES.md §0](https://github.com/PASAKON/Agents-Wikis/blob/main/IRON-RULES.md#section-0--read-the-wiki-first-when-uncertain-ceo-directive-2026-04-28) (`org:IRON-RULES.md`)** — *the* read-first rule
3. **[IRON-RULES.md §1](https://github.com/PASAKON/Agents-Wikis/blob/main/IRON-RULES.md#section-1--desk-model-one-agent-per-folder) (`org:IRON-RULES.md`)** — desk model + your role
4. **[IRON-RULES.md §23](https://github.com/PASAKON/Agents-Wikis/blob/main/IRON-RULES.md#section-23--engineering-integrity-ceo-directive-2026-04-29) (`org:IRON-RULES.md`)** — engineering integrity (NO MAGIC, VERIFY BEFORE DONE, DISSENT, NO SCOPE CREEP, TRACE BEFORE FIX)
5. **`projects/<your-folder>.md`** — the facts about the project you're sitting in
6. **`projects/mooniex-webapp-roles.md`** if you sit at a webapp desk — what your role's scope is
7. **`projects/mooniex-webapp-desk-<x>.md`** — your desk's specifics
8. Specific [`playbooks/`](../playbooks/) — only when you have a specific task

## Identify yourself (mandatory)

Per IRON-RULES §17 #1, every send needs a stable identity. Resolution rules:

| Role | `from_agent_id` resolution |
|---|---|
| Folder-bound DEV (Desk-A..F) | `claude-<sha256(host\|cwd\|day)>` — read from `~/.claude/office-state-<cwdHash>.json`. Auto-set by `office-pretool.mjs` on first edit. |
| CTO cross-desk | `claude-cto-cross-desk` (fixed) — when cwd is `/Users/gob/projects/LLMs/` |
| Secretary | Inherits CTO id but does **not** run the listener daemon (per §17 #8) |
| Claude Desktop user | `claude-desktop-<short_label>` — caller passes explicitly each send |
| curl / scripts | Pick a stable id like `script-<purpose>` and document it in your project file |

After Phase 1 of [`agent-listener.md`](agent-listener.md) lands, addressing switches to **`messaging_uuid`** (stable per role) — but the rule above stays for backward compat.

## First three commands by role

### If you sit at a webapp Desk (Senior Dev / ML / DevOps / QA / CTO)

```bash
# A. Verify desk + sync
basename "$PWD"
git fetch origin
git pull --ff-only

# B. Read what your role + desk owns
cat /Users/gob/projects/LLMs/projects/mooniex-webapp-roles.md
cat /Users/gob/projects/LLMs/projects/mooniex-webapp-desk-$(basename "$PWD" | sed -E 's/.*\(Desk-([A-Z]+)\)/\L\1/').md
# (the sed extracts e.g. "a" from "mooniex-webapp (Desk-A)")

# C. Run the local pre-deploy gates so you know baseline
pnpm tsc --noEmit
pnpm build
node scripts/lint-no-console.mjs
node scripts/lint-emoji.mjs
# All four must exit 0 before you ship anything.
```

### If you sit at `mooniex-option` / `mooniex-claudeflow` / other project

```bash
# A. Verify project + sync
basename "$PWD"
git fetch origin
git pull --ff-only          # or pull --rebase if branch convention says so

# B. Read the project file
cat /Users/gob/projects/LLMs/projects/$(basename "$PWD").md

# C. Read project's CLAUDE.md (the @-import surfaces wiki rules already)
cat CLAUDE.md
```

### If you are Secretary (LLMs scope only)

```bash
# A. Verify wiki is clean + synced
cd /Users/gob/projects/LLMs
git status --short
git pull --ff-only

# B. Read the latest audit + open questions
ls -lt AUDIT-*.md | head -1
cat AUDIT-2026-04-28.md          # or the most recent one

# C. Check inbox for CEO directives
# (use mcp__mooniex-coord__read_inbox or curl /api/office/messages/inbox)
```

## What NOT to do on your first turn

- ❌ Don't run `vercel deploy` until you've read [IRON-RULES §3](https://github.com/PASAKON/MoonieX-Wikis/blob/main/IRON-RULES.md#section-3--vercel-deploy-rules) (`mooniex:IRON-RULES.md`) end-to-end
- ❌ Don't write a migration until you've read [IRON-RULES §12](https://github.com/PASAKON/MoonieX-Wikis/blob/main/IRON-RULES.md#section-12--database-migrations-mooniex-webapp) (`mooniex:IRON-RULES.md`) + [`playbooks/database-migrations.md`](https://github.com/PASAKON/MoonieX-Wikis/blob/main/playbooks/database-migrations.md) (`mooniex:playbooks/database-migrations.md`)
- ❌ Don't add an env var until you've read [IRON-RULES §5](https://github.com/PASAKON/Agents-Wikis/blob/main/IRON-RULES.md#section-5--env--secrets) (`org:IRON-RULES.md`) (use `printf`, never `echo`)
- ❌ Don't broadcast to a channel without a `from_agent_id` resolved per the table above
- ❌ Don't claim "done" without the verification output (IRON-RULES §23.2)
- ❌ Don't bundle "while I'm here" changes into the asked task (IRON-RULES §23.4)

## When you genuinely don't know

The escape hatch is documented in IRON-RULES §0 — read the wiki, then DM the role owner via `mcp__mooniex-coord__send_message`. Do not guess.

## See also

- [`agent-messaging.md`](agent-messaging.md) — sending messages
- [`agent-listener.md`](agent-listener.md) — receiving messages (Phase 1+)
- [`local-dev-setup.md`](https://github.com/PASAKON/MoonieX-Wikis/blob/main/playbooks/local-dev-setup.md) (`mooniex:playbooks/local-dev-setup.md`) — getting the app running locally
- [`before-cross-folder-edit.md`](before-cross-folder-edit.md) — when your task touches another folder
- [IRON-RULES §0](https://github.com/PASAKON/Agents-Wikis/blob/main/IRON-RULES.md#section-0--read-the-wiki-first-when-uncertain-ceo-directive-2026-04-28) (`org:IRON-RULES.md`), [§1](https://github.com/PASAKON/Agents-Wikis/blob/main/IRON-RULES.md#section-1--desk-model-one-agent-per-folder) (`org:IRON-RULES.md`), [§17](https://github.com/PASAKON/Agents-Wikis/blob/main/IRON-RULES.md#section-17--agent-to-agent-messaging-office-mailbox) (`org:IRON-RULES.md`), [§23](https://github.com/PASAKON/Agents-Wikis/blob/main/IRON-RULES.md#section-23--engineering-integrity-ceo-directive-2026-04-29) (`org:IRON-RULES.md`)
