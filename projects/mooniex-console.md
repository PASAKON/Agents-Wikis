# MoonieX Console — mobile iTerm2 web console

**Status:** ✅ LIVE + VERIFIED 2026-07-23 — real `claude` CLI backend confirmed working on Contabo itself (clean banner, correct tab id, real cogitation indicator "✻ Sautéed for Ns", no garbled text). GH issue #31 (onboarding blocker) closed. Also merged+deployed 2026-07-24: back button (chat→session list), session list already shows open sessions for resume, cross-device resume, device-tagged grouping (auto 5-char ID, e.g. `iPhone #KS87U`), iOS scroll fix, `.claude/skills/` synced to Contabo. Next: CEO confirms all of this live on both iPhones.

**2026-08-07:** Fixed "UNKNOWN (N)" session-overview bucket — sessions started outside the Console (`spawn-cto.sh`/`spawn-cxo.sh` in the terminal) never got a `deviceLabel` row, since that's only ever written by the Console's own `createSession` flow. `bridge.js`'s `spawnAttach()` now backfills a box-derived label (`"Mac (terminal)"` / `"Contabo (terminal)"` via `os.hostname()`) the first time it notices one of these, without ever overwriting a real UI-set label. `src/tmux/{bridge,sessions}.js` + `src/db.js`, merged sha:22744a9 (task-81ebc285).
**Owner:** CTO · **Requested by:** CEO 2026-06-11
**Repo:** `PASAKON/MoonieX-Console` (own repo since 2026-08-03) · local `/Users/gob/Projects/mooniex-console`
**Bookmark:** `https://mooniex-contabo.tail400676.ts.net:8443/`
**Mockup:** `Agents/output/console-design/iterm-mobile-console.html`

### Split out of MoonieX-Agents — 2026-08-03
Extracted with `git filter-repo` (62% independent commit history), together with
`scripts/console-deploy.sh`, which only ever served it. Cut point tagged
`split/console-2026-08-03` in MoonieX-Agents. Runbook: `playbooks/repo-split-spinoff.md`.

Paths that used to read `console/…` are repo-root paths now — `console/src/tmux/command.js`
is `src/tmux/command.js`, and the deploy script sits at `scripts/console-deploy.sh`
inside the console repo.

**The contract is `ORG_ROOT`, and it is now required.** It used to default to `..`,
which only worked because the console sat one level inside the agents repo. The server
throws at boot if it is unset. The dependency points one way: the console spawns
`<ORG_ROOT>/scripts/cto-claude.sh`; nothing in MoonieX-Agents depends on the console.

**Latent deploy bug fixed in the same pass:** `console-deploy.sh` wrote
`ORG_ROOT=$REMOTE_DIR` into the box's `.env` — the console pointing at *itself*
(`/opt/mooniex-console`), where no launcher exists. The live box already held the
correct `/opt/mooniex-agents` (fixed by hand) and `stage_env` refuses to overwrite an
existing `.env`, so only a fresh deploy would ever have hit it. Now a named constant.
The split did not touch prod: service still `active`, `ORG_ROOT` still correct.

## Goal
Operator (CEO/dev) เปิด/ต่อ session ของ agent (CTO, CMO, CFO, CXO, …) จากมือถือผ่าน URL ที่ bookmark ได้ — cloud 100% ไม่พึ่ง Mac. **บรรลุแล้ว 2026-07-15** (chat loop). Terminal ต้องหน้าตา/พฤติกรรมเหมือน iTerm2 จริง — งานต่อเนื่อง 2026-07-17.

## LOCKED decisions
1. **Access = Tailscale-only** — tailnet `tail400676.ts.net` (iPhone 2 เครื่อง + Contabo `mooniex-contabo` 100.118.171.23 + Mac ทั้งหมด join แล้ว).
2. **P1 scope = iPhone 14 Pro + 14 Pro Max ก่อน** — verified.
3. Session runtime: **the real `claude` CLI** via `scripts/cto-claude.sh` (same launcher a Mac iTerm CTO tab uses) — changed 2026-07-17, was `runners.cto_chat` (a custom Python/rich REPL) until then. Runs on Contabo (org runtime migration — `decisions/2026-07-12-org-runtime-to-contabo.md`, Phase A-F ครบ).
4. **Console login แยกขาดจาก webapp `/admin`** — WebAuthn passkey + TOTP fallback, own SQLite DB, own cookie.
5. **Project availability = whatever's cloned onto Contabo, NOT GitHub presence.** A mobile session only sees/edits repos physically present on `/opt` or `/root/projects` on the Contabo box — pushing to GitHub alone does not make a project reachable from mobile. See "Project availability" section below.

## Architecture (live, 2026-07-17)
- **Host:** Contabo VPS, tailnet-only bind (`100.118.171.23:8443`), real Let's Encrypt cert via `tailscale cert`.
- **App:** `console/` (Node/Express + ws + node-pty), systemd `mooniex-console.service`, auto-restart + cert-renewal timer.
- **Org runtime:** `/opt/mooniex-agents` (repo + Python 3.12 venv), Claude Code CLI authenticated (long-lived OAuth token via systemd `EnvironmentFile`, inference-scope). Own independent `state/tasks.db` (not unified with Mac's — deliberate, see ADR Phase D).
- **Chat session command:** `console/src/tmux/command.js` → `bash <ORG_ROOT>/scripts/cto-claude.sh` — real `claude` CLI, not a custom REPL (see gotcha below for why this changed).
- **Auth:** first operator "CEO" registered via Face ID passkey (bootstrap flow) — confirmed working on iPhone directly (not cross-device/QR — that path hit a real bug, see Known gotchas).
- **Wiki tools:** gracefully report "not available" on Contabo (wiki content was deliberately never copied there — see ADR Phase C).
- **`.claude/skills/`:** synced to Contabo (2026-07-24) via narrow rsync include/exclude in `scripts/contabo-org-runtime-deploy.sh` (only `.claude/skills/**` passes; `.claude/settings*.json` and everything else under `.claude/` stays excluded) — project skills (`/session-open`, `/session-worktree`, `cto-merge-checklist`, …) now available from Contabo-run CTO sessions too.

## Project availability (which repos work from mobile) — added 2026-07-24

Console/mobile sessions run **on the Contabo box itself**, not the Mac. A repo
being on GitHub is not sufficient — it must be `git clone`d onto Contabo.
Canonical live copy of this table now also lives in the org repo's root
`CLAUDE.md` (auto-loads into every session, Mac or Contabo) so Claude always
knows this without needing a wiki call (which doesn't work from Contabo anyway).

**On Contabo today:**
| Project | Path on Contabo |
|---|---|
| mooniex-agents (org repo) | `/opt/mooniex-agents` |
| mooniex-console | `/opt/mooniex-console` |
| mooniex-claudeflow | `/root/projects/mooniex-claudeflow` |
| mooniex-option | `/root/projects/mooniex-option` |
| mooniex-alphatrader | `/root/projects/mooniex-alphatrader` |
| mooniex-line-automation | `/root/projects/mooniex-line-automation` |
| mooniex-line-poster | `/root/projects/mooniex-line-poster` |
| claude-usage-monitor | `/opt/claude-usage-monitor` |

**NOT on Contabo** (most notably `mooniex-webapp` — mobile cannot touch webapp code until it's cloned onto the box): mooniex-webapp, mooniex-genui, mooniex-moonx, mooniex-remotion, mooniex-wa-system, mooniex-company, mooniex-nohuman, mooniex-website-templete, mooniex-hyperframes, mooniex-claudesign, mooniex-controller-system, mooniex-scriptable.

**Maintenance rule:** whenever a repo is cloned onto (or removed from) Contabo, update both this table AND the root `CLAUDE.md` copy — keep them in sync.

## Known gotchas (2026-07-17)
- **Rendering didn't match real iTerm2 (found + fixed 2026-07-17):** the console was launching `python -m runners.cto_chat` — a bespoke hand-rolled REPL (Python `input()` + `rich.live.Live`) — instead of the actual `claude` CLI the CEO uses on Mac. Two concrete symptoms this caused: (1) no "thinking" indicator (that REPL never had one — it's a Claude Code CLI-native feature, not something a custom SDK script gets for free); (2) garbled/duplicated text, root-caused to `lib/logger.py`'s per-name logger being a process-wide singleton — `runners/cto.py` imports first and calls `get_logger("cto")` with the stdout-printing default ON, so when `cto_chat.py` later asked for the same logger with `stdout=False`, the early-return guard silently ignored that request and both the log line AND rich's Live render printed the same text. **Fix:** `console/src/tmux/command.js` now launches `scripts/cto-claude.sh` (the real CLI) instead. That script's `config/cto.mcp.json` reference was also hardcoded to the Mac dev path (`/Users/gob/Projects/Agents`) — broken on Contabo (`/opt/mooniex-agents`) — fixed by generating the MCP config at runtime from `$ROOT` instead of reading the static file. Dry-run on Contabo confirmed the real CLI's TUI renders cleanly through the tmux/node-pty/xterm.js bridge (theme picker screen showed correctly, no garbling).
- **One-time CLI setup step — RESOLVED 2026-07-23:** root's `claude` on Contabo needed one real interactive browser-authorize + code-paste to clear its first-run onboarding wizard (theme pick + login). Took several attempts across 2026-07-15 → 2026-07-23 because OAuth codes expire in minutes and kept going stale before being pasted back (tracked as GH issue #31, now closed). Verified directly on the box afterward: clean CLI banner ("Welcome back PASAKON!"), correct tab id, and a real test message got a proper response with the native cogitation indicator ("✻ Sautéed for 4s") — fully matching real iTerm2. **Still needed:** confirm the same thing through the actual Console UI on both iPhones (the fix only applies to sessions created from now on — any Console session created before 2026-07-17 is still running the old backend and should be recreated).
- **WebAuthn bootstrap bug (real, found live 2026-07-15):** `resolveOperatorForEnrollment` in `console/src/auth/routes.js` creates the operator DB row as soon as `/api/auth/webauthn/register-options` is called — **before** the registration ceremony actually completes/verifies. If the first passkey attempt is interrupted (e.g. user picks "use another device"/scans a cross-device QR and it fails, or backs out), the operator row is left behind with zero credentials, permanently closing `bootstrapOpen()` (`getOperatorCount() === 0`) with no way back in through the UI. Hit this live 2026-07-15 — CEO's first attempt via cross-device QR failed with "no passkey for this site" (correct behavior once bootstrap was wrongly closed), required a manual DB reset (`DELETE FROM operators/webauthn_credentials/totp_secrets WHERE id=...`) to reopen. **Not yet fixed in code** — worth a follow-up task to make operator creation atomic with credential verification (create only inside `register-verify`, not `register-options`). Workaround until fixed: always register directly on-device (Face ID, no cross-device/QR) on the very first attempt.
- Passkey registered via cross-device/QR risks not matching the device you actually want signed in — register directly on each iPhone you want to use, or rely on iCloud Keychain sync (same Apple ID) if that's confirmed working.
- `console.mooniex.com` pretty-domain DNS — still not done (needs GoDaddy DNS-01, deferred, not blocking).
- Org runtime credentials on Contabo: minimal by design — no GH token, no wiki SSH key, no company wiki content. Only the Claude Code OAuth token (inference-scope) exists there. See ADR for full reasoning.

## Follow-ups (not blocking, tracked here for later)
- Confirm the real-CLI rendering fix + back button + device-tagged session list on both iPhones via a freshly-created Console session.
- Fix the WebAuthn bootstrap-creation-order bug above.
- Decide if/when to unify `tasks.db` between Mac and Contabo (currently two independent instances, by design — ADR Phase D).
- `console.mooniex.com` custom domain (DNS-01 via GoDaddy API).
- Cross-project delegation from a Contabo-run CTO session (webapp/claudeflow/etc.) needs those repos + their push credentials separately — not built, not needed for the current use case.
- If CEO wants mobile sessions to read/write wiki directly, needs syncing LLMs wiki repo to Contabo + setting `WIKI_ROOT` there (not done — deliberate scope boundary, ADR Phase C). Flag before doing — expands what's on the box.
