# Cheatsheet — org essentials

> Copy-paste reference for what every agent does regardless of project:
> identity, wiki, messaging, and the integrity rules that bind every turn.
> Each command is hard-tested and quick-links to its canonical rule —
> **read the rule first**, then come back to this card.

> **MoonieX build/deploy commands are not here.** Webapp daily loop, Vercel
> deploy, env vars, migrations, Sentry, lint, and the test account live in
> [MoonieX's cheatsheet](https://github.com/PASAKON/MoonieX-Wikis/blob/main/CHEATSHEET.md)
> (`mooniex:CHEATSHEET.md`). Split 2026-08-03 per ADR 0013.

> Companion: [`playbooks/onboarding.md`](https://github.com/PASAKON/Agents-Wikis/blob/main/playbooks/onboarding.md) (`org:playbooks/onboarding.md`), [`playbooks/local-dev-setup.md`](playbooks/local-dev-setup.md). Linked from [`README.md`](README.md).

## Identity

```bash
# Canonical clone (no Desk-X suffix as of 2026-05-28)
basename "$PWD"                            # "mooniex-webapp" → role from Agents config
cat ~/.claude/office-state-$(printf '%s' "$PWD" | shasum -a 256 | head -c 12).json
# .agent_id is your from_agent_id (legacy — being phased out, see ADR 0007)
```

## Wiki

```bash
# Pull (SessionStart hook does this; manual when idle)
git -C /Users/gob/projects/LLMs pull --ff-only

# Find a rule
grep -n "^## Section" /Users/gob/projects/LLMs/IRON-RULES.md
grep -rn "<keyword>" /Users/gob/projects/LLMs --include="*.md"

# Inbound link check (§7 mandatory)
grep -rln "$(basename your-new-file.md)" /Users/gob/projects/LLMs --include='*.md'
```

## Messaging (MCP)

```
mcp__mooniex-coord__send_message(to_role="DevOps", body="...")
mcp__mooniex-coord__send_message(to_agent_id="<uuid>", replied_to="<msg_id>", body="...")
mcp__mooniex-coord__read_inbox(auto_ack=true)
mcp__mooniex-coord__read_thread(root_id="<uuid>")
mcp__mooniex-coord__list_active_agents()
```

`from_agent_id` resolves automatically from your office cache. Full rules: [IRON-RULES §17](IRON-RULES.md#section-17--agent-to-agent-messaging-office-mailbox).

## Office desk (LEGACY — being phased out)

The `/api/office/claim` endpoint and `webapp_office_*` tables are legacy
infrastructure from the multi-desk pattern (retired 2026-05-28). They still
function but no new code should depend on them. Role assignment now lives
in `Agents/config/projects.yaml`.

Full rules: [IRON-RULES §1](IRON-RULES.md#section-1--one-repo--one-folder-parallel-via-branches--gh-issues), [ADR 0007](https://github.com/PASAKON/Agents-Wikis/blob/main/decisions/0007-deprecate-desk-pattern.md) (`org:decisions/0007-deprecate-desk-pattern.md`).

## Engineering integrity (every turn)

- **NO MAGIC** — state assumptions, don't fabricate (§23.1)
- **VERIFY BEFORE DONE** — paste evidence, "should work now" forbidden (§23.2)
- **DISSENT** — blast radius + assumptions + reversibility before major change (§23.3)
- **NO SCOPE CREEP** — fix asked thing only (§23.4)
- **TRACE BEFORE FIX** — root cause, not symptom (§23.5)

Full text: [IRON-RULES §23](IRON-RULES.md#section-23--engineering-integrity-ceo-directive-2026-04-29).
