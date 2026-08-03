# Claude in Chrome — Operational Playbook

**Audience:** CTO, Tester, DevOps, Developer
**Last updated:** 2026-05-19 (CTO survey + API-first rule)
**Extension ID:** `fcoeoabgfenejglbffodgkkbkcdhcgfn` v1.0.70
**Logged-in account:** `pass.gob1@gmail.com`

---

## 0. API-first rule (CEO directive 2026-05-19)

**Hard rule:** before opening a browser tab via Claude in Chrome, check whether the target service exposes an API + a credential. If yes → use the API. Only fall back to the browser when no API path exists.

Decision tree per task:

```
need to talk to service X →
  is there a published API?  ─ no  ─→ use Claude in Chrome (sections 1+)
                             │
                             yes
                             ↓
  do we already have credentials? (env, secrets manager, MCP)
                             │
                             ├ yes ─→ call API with that credential
                             │
                             └ no  ─→ can we obtain a key without entering payment / SSO password?
                                       │
                                       ├ yes ─→ obtain key, then call API
                                       │
                                       └ no  ─→ use Claude in Chrome (sections 1+)
```

### 0.1 Why API first

- Faster (no page load / DOM scrape)
- Deterministic (no UI drift breaking parsers)
- Cheaper context (JSON vs. full page text)
- Auditable (one log line vs. screenshot trail)
- Doesn't depend on extension safety bucket — works on bucket-C services (xm, exness, stripe) if the service has an API and we have a token.

### 0.2 Known API surfaces (mooniex projects)

Quick lookup. Add new entries as we discover them. Source of truth lives next to each project's env documentation.

| Service | API endpoint | Credential env | Where used | Browser fallback |
|---|---|---|---|---|
| Supabase | `https://tlokhyqpthvxabweekps.supabase.co/rest/v1` + JS client | `SUPABASE_SERVICE_ROLE_KEY` (server) / `NEXT_PUBLIC_SUPABASE_ANON_KEY` (client) | webapp + claudeflow | supabase.com dashboard for logs/SQL editor |
| Vercel | `https://api.vercel.com/v*/...` | `VERCEL_TOKEN` (+ `VERCEL_TEAM_ID`, `VERCEL_PROJECT_ID`) | webapp deploys | vercel.com dashboard |
| Sentry | `https://sentry.io/api/0/...` | `SENTRY_API_TOKEN` (admin scope) or `SENTRY_API_TOKEN_RW` | webapp error audit | mooniex.sentry.io |
| Anthropic | `https://api.anthropic.com/v1/...` | `ANTHROPIC_API_KEY` | both | console.anthropic.com / platform.claude.com |
| OpenRouter | `https://openrouter.ai/api/v1/...` | `OPENROUTER_API_KEY` | both | openrouter.ai dashboard |
| Fal.ai | `https://fal.run/...` + `https://rest.alpha.fal.ai/...` | `FAL_API_KEY` (admin scope for billing) | claudeflow video pipeline | fal.ai dashboard |
| Deepgram | `https://api.deepgram.com/v1/...` | `DEEPGRAM_API_KEY` | claudeflow STT | deepgram.com |
| Notion | `https://api.notion.com/v1/...` | `NOTION_TOKEN` | claudeflow sync | notion.so |
| Google Drive | `googleapis.com/drive/v3` (OAuth2) | service-account / OAuth refresh token | claudeflow video manifest | drive.google.com (bucket B — needs Allow) |
| Gmail | `googleapis.com/gmail/v1` (OAuth2) | OAuth refresh token | mooniex-option channel bot | gmail.com (bucket B) |
| LINE Messaging | `https://api.line.me/v2/...` | channel access token | claudeflow webhooks | developers.line.biz console |
| Meta Graph | `https://graph.facebook.com/v*/...` | page / app access tokens | claudeflow FB+IG | business.facebook.com |
| Telegram Bot | `https://api.telegram.org/bot<token>/...` | bot token | claudeflow telegram | t.me (bot pages only) |
| Postforme | `https://api.postforme.dev` | `POSTFORME_API_KEY` (bearer) | webapp social connections | app.postforme.dev |
| Twelve Data | `https://api.twelvedata.com/...` | `TWELVE_DATA_API_KEY` | webapp DASH-Q3 quote feed | twelvedata.com |
| GitHub | `gh` CLI + REST | `gh auth login` per machine | both | github.com |
| XM (broker) | Partner API — `XM_API_TOKEN` exists in claudeflow `.env` | `XM_API_TOKEN` | claudeflow balance feeds | **none** — `*.xm.com` is bucket C, never via browser |
| Exness (broker) | Partner API — credentials in claudeflow `.env` | Exness partner creds | claudeflow | **none** — `*.exness.com` is bucket C |
| Stripe | `https://api.stripe.com/v1/...` | `STRIPE_SECRET_KEY` (when issued) | webapp billing | **none** — `stripe.com` is bucket C |
| Omise | `https://api.omise.co` | `OMISE_SECRET_KEY` | webapp billing | omise.co marketing site (bucket A) but dashboard ops via API only |

### 0.3 When API path is forbidden / inadvisable

- **You're the CTO doing UI/UX visual QA.** The whole point is to look at the rendered page; API short-circuits that. Stay on the browser path.
- **You're verifying a non-engineering surface** — pricing layout, copywriting, image alt text, hero CTA visibility. Browser only.

### 0.4 Acquiring an API key when none exists (CTO-exclusive)

When §0 decision tree lands on "no credential, no easy way to obtain via API" **and** the missing key is from a service where the CEO has an active logged-in session in Chrome — the CTO MAY use Claude in Chrome to navigate to the service's dashboard, generate a new API key, and persist it.

**Hard rules:**

1. **CTO-only operation.** No DEV role (tester / developer / devops / web_designer / security / data_analyst / prompt_engineer) is allowed to drive a key-acquisition flow. They have no Claude in Chrome tools provisioned. If a DEV needs a key, they file a blocker → CTO handles in a separate turn.
2. **Bucket constraint applies.** The service's dashboard must be bucket A or B per §1–§2. Bucket C (broker, payment, banking) cannot be used to obtain keys via Claude in Chrome regardless of CEO presence — by Anthropic safety policy.
3. **Save once, reuse forever.** Newly acquired keys are written to the **project's `.env` (VPS or local)** AND mirrored to the relevant secrets store (Vercel env, Supabase secret, or GitHub Actions secret per project). After persistence, the CTO must not re-request the key. Future tasks read from env. Re-acquiring an already-present key is a violation.
4. **CTO never types secret characters into a Claude-in-Chrome browser session.** Generation is performed by the dashboard UI (a click that the dashboard then renders); the CTO captures the rendered key string via `get_page_text` / `javascript_tool`, then writes it to env. If the dashboard requires the CTO to TYPE a name/label for the key, that's fine — names are not secrets.
5. **Audit trail required.** Every newly-acquired key gets a wiki note (`playbooks/secrets-rotation.md` or per-project env doc) with: service, scope, date acquired, target env(s), and the task_id that needed it. No silent key creation.

### 0.5 CTO revocation right

The CTO retains the right to revoke any API key the org owns when:

- A key is suspected leaked (committed to git history, posted to a public channel, surfaced in a screenshot).
- A key has scope wider than the current need (e.g., admin-scope where a read-only would suffice — narrow and re-issue).
- A key has been unused for ≥ 90 days (rotation hygiene, opt-in per §0.7).
- The service's vendor flagged the key in a breach notification.

**How:** call the service API (preferred) or use Claude in Chrome to the dashboard's "API tokens" page → revoke → confirm. Then rotate any consumers (Vercel env, `.env`, GitHub secret) atomically — never leave a half-rotated state.

### 0.6 Don't revoke too often (CEO directive)

Revocation is a privilege, not a habit.

- ❌ Don't revoke on suspicion alone — confirm leak with `git log -p --all -S <key-fragment>` or vendor breach feed first.
- ❌ Don't rotate every key every session as a precaution — burns key-management overhead with no security gain.
- ❌ Don't revoke a key because a task failed; failures rarely trace to credential compromise.
- ✅ Revoke immediately on confirmed leak.
- ✅ Coordinate planned rotations with CEO at most once per quarter unless a service requires more.

If in doubt, file a wiki ADR under `decisions/` proposing the revoke + reason before acting. CEO can veto within 24h.

### 0.7 Forbidden

- ❌ Asking the CEO to type credentials into a browser when an env var with the same scope already exists in the project's `.env`. Use the env var.
- ❌ Embedding API keys into URLs, screenshots, or commit messages.
- ❌ Calling an unknown / unreviewed third-party API endpoint because "the page mentioned it" — confirm in §0.2 or the project's env docs first.
- ❌ Re-acquiring a key that already lives in env. Read once, reuse.
- ❌ DEV roles acquiring keys via any path. Always CTO + browser dashboard.
- ❌ Acquiring or revoking keys for bucket-C services (xm, exness, stripe) via Claude in Chrome — impossible by design, do not attempt.

### 0.8 Proactive offer rule (CEO directive 2026-06-11)

CEO: "ไม่อยากมาบอกบ่อย ๆ ว่าอนุญาตให้ใช้ Chrome — อยากให้ Claude พิมพ์ขออนุญาตมาเอง"

Whenever a task surfaces a **manual external action** that the CEO would otherwise do by hand in a web dashboard (DNS record, domain config, dashboard toggle, form fill, OAuth consent, account setting), the CTO must NOT just hand the CEO a step list. Instead, in the same message:

1. Run the §0 decision tree first — API path wins if a credential exists.
2. If the fallback is a browser and the domain is bucket A/B → **proactively offer**: one line "จะให้ผมใช้ Chrome ทำ X ให้เลยไหม?" + a 1-line plan (site, exact change, what login state is needed).
3. CEO answers with any short approval token — ได้ / yes / no / approve / go / OK — that single word IS the authorization for that specific offered action. No re-asking, no broader scope than offered.
4. Bucket C (broker / payment / banking) — never offer; state the manual steps instead.
5. If the action needs the CEO's logged-in session and it isn't present, say so in the offer ("ต้องล็อกอิน GoDaddy ค้างไว้ใน Chrome ก่อน").

Manual step lists are now the fallback for bucket-C / no-session cases only.

---

## 1. Three permission states

When `navigate(url)` is called, the extension classifies the target domain into one of three buckets:

| State | Error / behavior | Override path |
|---|---|---|
| **A. Auto-allowed** | Navigation succeeds, no popup | None needed |
| **B. permission_required** | Returns `permission_required: <domain>` error. Chrome shows in-page Allow popup. After CEO clicks Allow → domain added to "Your approved sites" → subsequent navigates succeed | CEO clicks Allow once per domain |
| **C. Safety-restricted** | Returns `This site is not allowed due to safety restrictions.` **No popup.** Hardcoded Anthropic policy (broker / payment / financial / consumer accounts) | **None. By design.** Cannot whitelist. |

Decision tree for any new domain:
```
navigate(url) →
  → success            → bucket A
  → permission_required → bucket B (Allow popup needed)
  → safety restrictions → bucket C (won't fix; switch stack)
```

---

## 2. Domain map for mooniex projects (tested 2026-05-19)

### A. Auto-allowed (use directly)

| Domain | Used by | Purpose |
|---|---|---|
| `localhost:3000` | webapp + claudesign | Local dev preview |
| `localhost:3001` | claudeflow | Local webhook server |
| `github.com` (in approved list) | all | Issues, PRs, repo browsing |
| `www.mooniex.com` | webapp | Production frontend |
| `supabase.com` | webapp + claudeflow | DB dashboard, SQL editor, logs |
| `vercel.com` (`/passgob1-8454s-projects`) | webapp | Deploy logs, env vars, analytics |
| `mooniex.sentry.io` | webapp | Error grouping, session replay |
| `notion.so` | claudeflow | Notion ↔ Supabase sync verification |
| `console.anthropic.com` → `platform.claude.com` | both | API usage, billing |
| `openrouter.ai` | both | Model usage, billing |
| `fal.ai` | claudeflow | Image gen + TTS credits / dashboard |
| `omise.co` | webapp (billing) | Marketing site (transactions page may differ) |
| `developers.line.biz` | claudeflow | LINE Channel + Login config |
| `business.facebook.com` (Meta Business Suite) | claudeflow | FB / IG webhook + Page asset config |
| `t.me` | claudeflow | Telegram bot pages |
| `google.com` (search) | all | Generic search |

### B. permission_required (CEO Allows once)

| Domain | Used by | Why blocked initially |
|---|---|---|
| `drive.google.com` | claudeflow video pipeline | OAuth-protected user data — Anthropic gates Google services behind explicit Allow |
| `mail.google.com` (expected) | mooniex-option Gmail bot | Same as Drive |
| `calendar.google.com` (expected) | future | Same |

**Procedure:** Tester runs `navigate(target)` → returns `permission_required` → pings CEO via terminal/message → CEO clicks Allow in browser popup → Tester retries.

### C. Safety-restricted (NEVER works via Claude in Chrome — use Playwright or manual)

| Domain | Used by | Why |
|---|---|---|
| `*.xm.com` (`mypartners.xm.com`, `my.xm.com`, `www.xm.com`) | claudeflow XM token, mooniex-webapp broker integration | Broker / trading |
| `partners.exness.com`, `my.exness.com` | claudeflow Exness creds | Broker |
| `stripe.com`, `dashboard.stripe.com` | webapp billing | Payment processor |
| `my.line.me` | (LINE user consumer site) | Consumer account |
| `trade.mql5.com` (expected) | MT5 broker page | Trading platform |
| Banking / SSO password pages | any | Credentials entry |

**No bypass attempts.** Redirect through Google, archive.org, proxies — all blocked because filter checks **final destination domain**. Per Anthropic safety policy.

**Workaround:** for any task involving bucket C:
1. Manual: CEO opens page, screenshots → AI reads screenshot.
2. Playwright local: separate Python script under `/Users/gob/Projects/Agents/scripts/` runs against user's Chrome profile (not subject to extension safety filter).

---

## 3. Use cases per role

### CTO Tester (UI/UX + functional QA)

**Allowed scope:** bucket A domains only (or B after Allow).

Standard journeys:
- **webapp homepage smoke** — `https://www.mooniex.com` → verify hero loads, no console errors, primary CTA clickable
- **webapp local dev verify** — `http://localhost:3000` → after dev push, confirm hot-reload diff matches PR
- **Sentry latest errors** — `https://mooniex.sentry.io/issues/?statsPeriod=24h` → list last 24h issues for CTO triage
- **Vercel build status** — `https://vercel.com/passgob1-8454s-projects/mooniex-webapp/deployments` → confirm last deploy = Ready
- **Supabase log spot-check** — `https://supabase.com/dashboard/project/tlokhyqpthvxabweekps/logs/postgres-logs` → grep recent errors

Tools per task: `navigate` + `get_page_text` + `javascript_tool` (for DOM assertions) + `read_console_messages` (for JS errors).

### CTO DevOps / Developer (config + API surface)

**Allowed scope:** bucket A. Bucket B with CEO co-pilot on call.

Standard journeys:
- **Vercel env var audit** — list all env vars + flag echo-tainted (`\n`-suffixed) values
- **Supabase migration verify** — SQL editor → run `select * from webapp_schema_migrations order by applied_at desc limit 10`
- **LINE channel config diff** — `https://developers.line.biz/console/channel/<id>` → compare webhook URL + reply token settings
- **Meta webhook subscription check** — `business.facebook.com` → IG webhook subscription state
- **Anthropic / OpenRouter usage** — pull spend, model breakdown for cost report

**Out of scope for Claude in Chrome:** anything in bucket C — broker tokens, Stripe dashboard, payment dispute resolution. Hand back to CEO.

### Data validation

- **Supabase row spot-check** — table editor or SQL — confirm webhook write landed (e.g. `webapp_user_activity_log` count delta)
- **Production endpoint health** — `navigate(https://www.mooniex.com/api/health)` → assert response body JSON shape
- **Notion sync correctness** — open Notion page → diff against Supabase `posts` row

---

## 4. Standard tool sequence

```
1. tabs_context_mcp                         # know what tabs exist
2. tabs_create_mcp                          # fresh tab per task
3. navigate(url, tabId)                     # bucket A/B
   ↳ permission_required → notify CEO → wait for Allow → retry
   ↳ safety restrictions → STOP — switch to Playwright/manual
4. get_page_text(tabId)                     # primary read
   OR read_page(tabId, filter:"interactive") # for click targets
5. javascript_tool(text:"...", tabId)       # DOM assertions / extract structured data
6. read_console_messages(tabId, pattern)    # JS error scan
7. Report findings → close tab or leave for CTO follow-up
```

`browser_batch` for predictable multi-step (navigate → click → type → screenshot). Stops on first error — only batch when each step is expected to succeed.

**Avoid:** triggering JS alerts/confirms (they block the extension). Use console.log + `read_console_messages` instead.

---

## 5. Out of scope (do not attempt)

- Logging into broker / trading platforms — bucket C, by design
- Entering credit card numbers, account numbers, OTP, API keys — Anthropic privacy policy
- Bypassing safety filter via redirect, archive, proxy, URL encoding — policy violation
- Modifying browser extension settings or `managed_schema.json` to circumvent — security boundary

---

## 6. Related

- Issue [`PASAKON/mooniex-agents#3`](https://github.com/PASAKON/mooniex-agents/issues/3) — original XM balance test that triggered this survey
- [`playbooks/browser-agent-handoff.md`](browser-agent-handoff.md) — when bucket C forces handoff to Playwright/manual
