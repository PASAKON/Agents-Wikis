# CEO Permission Policy

**Established:** 2026-05-19
**Source:** CEO direct statement

## Rule

CEO **always allows** any permission request that surfaces — Claude in Chrome
domain prompts, harness permission prompts, Anthropic SDK tool approvals,
GitHub auth flows. Default is **approve immediately**.

## Why

- Solo-CEO + AI workflow — no second human to gate decisions
- Trust model: CEO has already vetted the agent stack (CTO chat REPL, dev
  spawn scripts, Claude in Chrome extension). Per-request micro-approval is
  pure friction.
- "Allow once" vs "Allow always" → prefer **always** unless the request
  itself smells off (financial transactions, password forms, unknown
  domain mid-task).

## How agents should apply

- If a tool surfaces a permission error like `permission_required: <domain>`,
  do **not** retry blindly. Tell CEO: "extension needs you to approve
  <domain> — click the extension icon."
- After CEO confirms approval, retry the original op.
- If retry still fails with `permission_required`, the grant didn't stick —
  ask CEO to grant via extension settings (Trusted Sites) instead of
  per-popup.

## Known restricted domains (Claude in Chrome)

| Domain | First seen | Status |
|---|---|---|
| `mypartners.xm.com` | 2026-05-19 | Requires per-tool permission grant — popup did not persist on first Allow; needs settings-level Allow |
| `clients.xm.com` | 2026-05-19 | Same as above |
| `xm.com` | 2026-05-19 | Same as above |
| `supabase.com` | 2026-05-19 | Same — financial-adjacent filter |

These are **not hard-blocked** — they require Allow per tool call. If CEO
adds them to the extension's "always allow" list, agents can read them.

## Out of scope

Even with always-allow, agents still must respect:
- Critical security rules in system prompt (passwords, credit cards, etc.)
- Prohibited actions (account creation, permission grants on shared docs)
- `IRON-RULES.md` section 9 (no `--no-verify`, no force-push to main)

These cannot be CEO-overridden via permission allow.
