# Wiki sync (mandatory)

A central knowledge wiki lives at `/Users/gob/projects/LLMs/`. It documents every Mooniex project's rules, deploy procedure, env handling, and cross-folder dependencies — readable by every other DEV (Claude / Codex agent).

## Read

Before starting work that touches more than one folder, or before a `vercel deploy` / `git push`:
- `/Users/gob/projects/LLMs/INDEX.md` — project matrix (start here)
- `/Users/gob/projects/LLMs/projects/<this-folder>.md` — your project's facts
- `/Users/gob/projects/LLMs/IRON-RULES.md` — canonical rules (verbatim quotes from every CLAUDE.md / AGENTS.md / setup-windows.md)
- `/Users/gob/projects/LLMs/playbooks/before-cross-folder-edit.md` if editing another folder

## Read FIRST when uncertain (CEO directive 2026-04-28)

**Whenever you can't find information, don't know if your approach is correct, or aren't sure your work aligns with project direction → read the wiki BEFORE asking the user, BEFORE guessing, BEFORE running commands.** This is mandatory, not optional.

Triggers (any one is enough):
- You searched the codebase and didn't find the answer
- You're about to write a `console.error` / emoji / migration / cron / route — pattern unclear
- You don't know which desk / folder owns this scope
- You don't know which env var, secret rotation cadence, or API contract applies
- You're about to handle an incident and aren't sure of the runbook
- A user request seems to conflict with a project rule you remember vaguely

Order to consult:
1. `IRON-RULES.md` (search by section number — `## Section N`)
2. `playbooks/<topic>.md` (deploy, secrets-rotation, cross-folder-edit, agent-messaging, …)
3. `projects/<folder>.md` (per-project facts)
4. `INDEX.md` (if you don't know which project file to open)
5. `CROSS-FOLDER-MAP.md` (if work spans folders)

Only after exhausting the wiki: ask the user, or send a DM via `mcp__mooniex-coord__send_message` to the CTO / role owner.

Why: every "I forgot the rule" → "I deployed a thing that broke prod" trace in the audit log started with someone skipping this step. The wiki exists so the same incident isn't relived per session.

## Engineering integrity (CEO directive 2026-04-29 — IRON-RULES §23)

Three rules every agent follows on every turn — non-negotiable.

### NO MAGIC — never guess
All assumptions explicit. If context is missing, state assumptions. Don't hallucinate hidden infra or invent unspecified services. Say `"I'm assuming X because Y"` and verify before acting.

### VERIFY BEFORE DONE — evidence, not assertions
Never claim a change is complete without running verification. **The phrase "should work now" is forbidden.**
- ✅ `"Edited X. Ran <command> — exit 0, output: <paste>"`
- ❌ `"Done — should work now"`

For deploys: paste prod URL + smoke-test response, not just "promoted". For docs: paste the diff or `git show <hash>`.

### DISSENT — argue before commit on major changes
Before any major change (migration affecting > 1 table, prod deploy that changes behavior, env var add/remove, cron change, architecture pivot, cross-folder edit, `vercel deploy --prod`), state at least 2 of:
- What's the blast radius?
- What assumptions are we making?
- What's the reversibility path?
- What are we NOT seeing because of momentum?

Dissent is a record, not a veto. User can override; the trace stays.

### NO SCOPE CREEP
Fix only what was asked. Don't bundle "while I'm here, I'll also reformat / refactor". Spot related work → log it (`MIGRATIONS.md` or spawn-task), ship the original ask first.

### TRACE BEFORE FIX
Identify the **root cause** before patching. If you can't reproduce: say so. Symptom-only fixes accumulate into the same "I forgot the rule" debt §0 was created to prevent.

Full text: [IRON-RULES.md §23](https://github.com/PASAKON/Agents-Wikis/blob/main/IRON-RULES.md#section-23--engineering-integrity-ceo-directive-2026-04-29) (`org:IRON-RULES.md`).

## Secrets / API keys — check the registry FIRST (IRON-RULES §34)

Before you create OR rotate any API key / secret / token:
- Read [`playbooks/api-key-registry.md`](https://github.com/PASAKON/MoonieX-Wikis/blob/main/playbooks/api-key-registry.md) (`mooniex:playbooks/api-key-registry.md`) — every key, its purpose, owner, and which env var holds it in webapp vs claudeflow.
- **A key for your purpose almost always already exists — reuse it.** Don't mint a duplicate. ("Allowed to use Claude-in-Chrome" ≠ "create a new key" — verify the existing one first.)
- One logical secret can have different names per repo (e.g. `SUPABASE_SERVICE_ROLE_KEY` in webapp == `SUPABASE_SERVICE_KEY` in claudeflow). Rotate ALL stores together, or you get "works in prod, 401 locally".
- Every key change = append a Changelog line in the registry (date · key · who · why) in the same commit. No silent changes.

## Update (your responsibility)

You own the wiki entry for the project you're editing. Update wiki **before** `git push`, in the same change set.

| You changed | Update |
|-------------|--------|
| This project's `CLAUDE.md` / `AGENTS.md` / `setup-windows.md` | `LLMs/projects/<this>.md` + relevant verbatim section in `LLMs/IRON-RULES.md` |
| New webhook / env share / table FK to another project | `LLMs/CROSS-FOLDER-MAP.md` (mermaid edge + edge-detail section) |
| New folder under `/Users/gob/projects/` | Create `LLMs/projects/<new>.md` + row in `LLMs/INDEX.md` |
| Postmortem produced a new rule | Add to `LLMs/IRON-RULES.md` + "Recent incidents" in project file |

Do not paraphrase verbatim quotes — re-pull from source on each update.

Full ownership matrix: `LLMs/MAINTAINERS.md`.

## Skip wiki update for

- In-progress task state, today's TODOs
- Pure code changes that don't change a documented rule
- Bug fixes that don't change the rule itself
