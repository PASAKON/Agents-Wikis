# Playbook — Browser agent handoff (captcha / login / 2FA)

**Created:** 2026-05-18
**Last rewrite:** 2026-05-19 (switched from Auto Browser → Claude in Chrome)
**Owner:** CTO
**Applies to:** any CTO / DEV session that needs a real browser

---

## 1. Why

UI / UX tests, form fills, and any flow behind a login eventually hit a
blocker the AI cannot solve alone:

- captcha (reCAPTCHA, Cloudflare, hCaptcha)
- 2FA (SMS code, TOTP, push approval)
- WebAuthn / passkey
- consent / cookie banners that change layout
- session expiry mid-test

The reliable pattern is **AI works, AI pauses, human takes over in the
same live browser, AI resumes**. Claude in Chrome (the Anthropic
extension) gives the AI direct control of the CEO's real Chrome and
pauses automatically on login / captcha screens — the CEO solves the
blocker in the same tab the AI is using and the AI resumes.

## 2. Why Claude in Chrome (not Auto Browser)

We started with [Auto Browser](https://github.com/LvcidPsyche/auto-browser)
(Docker + Playwright + Chromium + noVNC) on 2026-05-18. Worked end-to-end
against example.com, but for the typical solo CEO + AI flow it was
overkill:

| | Claude in Chrome | Auto Browser |
| --- | --- | --- |
| Saved passwords / sessions | ✅ Chrome profile inherited | ❌ fresh Chromium per session |
| Setup | `claude --chrome` or `/chrome` | docker + clone + venv + ALLOWED_HOSTS env |
| Captcha pause | ✅ built-in | ✅ via `request_human_handoff` |
| RAM footprint | ~0 (uses existing Chrome) | ~500 MB-1 GB per container |
| Multi-DEV parallel | ❌ 1 connection per Chrome instance | ✅ designed for it |
| Trust model | sees CEO's real logins | isolated test profile |

For the org's current workload (1 CEO + 1 active AI session at a time)
Claude in Chrome wins. Auto Browser is the right answer only when we
need many AI agents driving browsers in parallel without touching the
CEO's real session — we'll revisit it when that workload appears.

Auto Browser stack removed 2026-05-19:
- docker compose stopped + containers removed (`auto-browser-controller-1`,
  `auto-browser-browser-node-1`)
- `Agents/config/dev-browser.mcp.json` deleted
- `Agents/runners/dev_init.py` BROWSER_ROLES branch reverted (all DEVs
  share `dev.mcp.json` again)
- `/Users/gob/Projects/auto-browser/` clone kept on disk; CEO can
  `rm -rf` it when done auditing

## 3. Bring-up

```bash
# In any Claude Code session that needs the browser:
/chrome                      # toggle on for this session
# or launch with it pre-enabled:
claude --chrome
```

Prerequisites (one-shot per laptop):

- Google Chrome or Microsoft Edge installed
- [Claude in Chrome extension](https://chromewebstore.google.com/detail/claude/fcoeoabgfenejglbffodgkkbkcdhcgfn)
  v1.0.36+
- Claude Code v2.0.73+
- Direct Anthropic plan (Pro / Max / Team / Enterprise)

If Claude Code reports "Chrome extension not detected", restart Chrome
then re-run `/chrome` → Reconnect.

## 4. Standard flow

```
Claude (in this session) opens a Chrome tab and starts the work.
Claude inherits the CEO's existing login state — Gmail, GitHub,
saved passwords, etc.
   ↓
If Claude hits a login form / captcha / 2FA prompt, it pauses
automatically and tells the CEO what to do.
   ↓
CEO switches to that Chrome tab, completes the blocker.
   ↓
Claude detects the resolved state and continues.
```

No noVNC, no docker, no separate Chrome window — the AI works in the
CEO's regular Chrome.

## 5. Handoff protocol

Claude in Chrome already pauses on its own. The `request_human_handoff`
MCP tool from `Agents/runners/dev_mcp_server.py` is still available for
**non-Chrome scenarios**: a DEV running headless (no Chrome) that needs
the CEO to do something outside the AI's reach. Example uses:

- "this private repo needs an SSH key the CEO holds"
- "deploy is blocked on a Vercel approval the CEO has to click"
- "we need a webhook secret pasted into the env file"

```python
mcp__org__request_human_handoff(
    kind="other",                 # or login/2fa/auth_refresh/captcha/consent/webauthn
    hint="add VERCEL_TOKEN to mooniex-webapp .env then reply `resume`",
    page_url="",                  # optional
    novnc_url="",                 # optional, empty when not browser-based
    session_id="",                # optional
)
```

Side effects unchanged: GH issue + status flip to `blocked_human` +
prominent line in CTO chat.

The captcha/login wording is still relevant — DEVs in headless E2E
runs (Playwright MCP in `dev.mcp.json`) hit captchas too and use the
same tool to ask the CEO to switch to a Chrome tab and solve.

## 6. Auth persistence

Claude in Chrome uses the CEO's Chrome profile, so the CEO's
already-logged-in state for every site (Gmail, GitHub, broker dashboards,
Notion, Vercel, etc.) is automatically available. There is **no
`save_auth_profile` step** like Auto Browser had — Chrome's own session
cookies persist normally.

If a site logs the CEO out, the next AI task simply pauses on the login
form and asks the CEO to sign in once.

## 7. Watchdog rules (unchanged)

- `blocked_human` excluded from the 30-min stall check.
- After `HUMAN_TIMEOUT_S = 24h` in `blocked_human`, watchdog escalates
  to `stalled` + GH issue (CEO assumed unavailable).
- `tasks.pid` gate still protects against killing the wrong tab.

## 8. Safety rails

- Claude in Chrome inherits the CEO's full Chrome session → high trust.
  Do not run untrusted prompts that could be used to exfiltrate
  account data. Site permissions are managed in the extension itself
  (`chrome://extensions` → Claude → site allowlist).
- For untrusted-prompt or multi-tenant scenarios, fall back to
  Playwright MCP with `storageState` (separate test-user profile).
- Reference: [Use Claude Code with Chrome (beta)](https://code.claude.com/docs/en/chrome)
  and [Piloting Claude in Chrome](https://www.anthropic.com/news/claude-for-chrome).

## 9. Out of scope (file separate playbooks if needed)

- Multi-DEV parallel browser farms (revisit Auto Browser when this
  workload appears)
- E2E test runs that should not see CEO's logins (use Playwright with
  per-suite storageState instead)

## 10. Changelog

| Date | Author | Change |
| --- | --- | --- |
| 2026-05-18 | CTO | Initial draft using Auto Browser stack. Wiring committed in Agents (`dev-browser.mcp.json`, dev_init.py role branch). Docker bring-up successful, E2E on example.com green. |
| 2026-05-18 | CTO | `mcp__org__request_human_handoff` MCP tool added (commit 82243be). §6 rewritten around the new tool. |
| 2026-05-19 | CTO | **Rewrite.** Removed Auto Browser path (docker stopped, `dev-browser.mcp.json` deleted, BROWSER_ROLES branch reverted). Switched to Claude in Chrome for solo CEO + AI workflows. `request_human_handoff` kept — still useful for non-Chrome handoffs (deploy approvals, SSH keys, env updates). |
