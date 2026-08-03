# Reference — Supabase project registry (which project lives where)

Living registry. Update whenever a Supabase project/org/account is added,
renamed, or moved. Companion to [ADR 2026-08-02 — account/org/project quota audit](https://github.com/PASAKON/MoonieX-Wikis/blob/main/decisions/2026-08-02-supabase-account-org-quota-audit.md) (`mooniex:decisions/2026-08-02-supabase-account-org-quota-audit.md`)
— the "why": real free-tier cap is 2 projects **per account**, not per org.

## Registry

| App | Account (email) | Org name | Project name | Project-ref | MCP server name |
|---|---|---|---|---|---|
| MoonieX (webapp + claudeflow, shared) | pass.gob1@gmail.com | MoonieX | MoonieX (default) | `tlokhyqpthvxabweekps` | `supabase` |
| LungNote | pass.gob1@gmail.com | LungNote | LungNote Project | `qkaxvockysyazmtormvf` | `supabase-lungnote` |
| Chatudo | pass.gob2@gmail.com | Chatudo | Chatudo Project | `gcjdirrtvybhvghjtscf` | `supabase-chatudo` |
| LinkReed | pass.gob2@gmail.com | LinkReed | LinkReed Project | `tlkujtschyhzuwumddej` | `supabase-linkreed` |

All 4 wired in `Agents/.mcp.json` as of 2026-08-03 — no more manual SQL via
Chrome dashboard needed for any of these.

**Not in active MCP rotation** (exist on pass.gob1, out of current scope):
- Org `arna-ai` (1 project) — pre-existing, purpose not documented here
- Org `ClaudeFlow` (0 projects, empty)

`mooniex-project3` — the empty org created live during the 2026-08-02 quota
test — was deleted 2026-08-03 (confirmed cleaned up, back to the original 4
orgs on pass.gob1).

## Access tokens (Personal Access Tokens, one per Supabase *account* not per project)

| Account | Env var | Where set | Expiry |
|---|---|---|---|
| pass.gob1@gmail.com | `SUPABASE_ACCESS_TOKEN` | `Agents/.claude/settings.local.json` (gitignored) | pre-existing, unknown expiry |
| pass.gob2@gmail.com | `SUPABASE_ACCESS_TOKEN_GOB2` | `Agents/.claude/settings.local.json` (gitignored) | **Never** (set 2026-08-03) |

A PAT authenticates for every org/project the account owns — one token per
account covers all its projects (confirmed: `SUPABASE_ACCESS_TOKEN` alone
covers both MoonieX and LungNote, two different orgs, same pass.gob1 account).

## MCP wiring

Server definitions live in `Agents/.mcp.json`; each must also be listed in
`enabledMcpjsonServers` in `Agents/.claude/settings.local.json` or Claude Code
won't load it. **New/changed server entries require a session restart** to
take effect — they don't hot-reload mid-session.

## Naming convention (keep this when adding a 5th+ project)

- Org name = app name, capitalized, no suffix (`Chatudo`, not `chatudo-org`)
- Project name = `<AppName> Project` (matches what's already live for all 4)
- MCP server name = `supabase-<lowercase-app-name>` (e.g. `supabase-chatudo`)
- Env var for a new account's token = `SUPABASE_ACCESS_TOKEN_<SHORT_TAG>` (e.g. `_GOB2`)
