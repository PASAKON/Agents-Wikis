# Playbook — Cron rules (universal)

> Owner: CTO. Last refresh 2026-04-28.
> Scope: every Mooniex project that ships a scheduled job.
> One scheduler, predictable cost, no fan-out abuse.

## Why this exists

Every scheduled job costs something — function invocations, DB
reads/writes, external API hits, email sends. Three rules
killed past incidents:

1. **One scheduler.** Two schedulers = double-firing + drift +
   "wait, who owns this?". On 2026-04-28 we found a duplicate
   pg_cron prune running alongside `/api/cron/api-usage-prune`.
   Same delete twice a day, race-y, hard to attribute cost.
2. **Cadence floor.** A cron firing every 1s = a stream, not a
   cron. Use the right tool (Realtime / queue / SSE).
3. **Fan-out cap.** A weekly digest that tries to email 50 k
   free users is a spam complaint waiting to happen + email
   provider blacklist + IP reputation tank.

## Hard rules (every project, every agent)

### R1. Vercel is the only scheduler

- All schedules live in `vercel.json` `crons[]`.
- Forbidden in source code:
  - `pg_cron` / `cron.schedule` in Supabase migrations
  - `node-cron`, `croner`, any node package that schedules
  - `setInterval` longer than 60s in long-running processes
  - `setTimeout` chains that recursively schedule themselves
- Exception: workflow_dispatch GitHub Actions (manual triggers)
  and Supabase Edge Functions are out of scope here — they're
  one-off, not periodic.

### R2. Cadence floor: 1 minute

- Tightest schedule: `*/1 * * * *` (every minute).
- Anything tighter = streaming/realtime requirement, not cron.
  Use Supabase Realtime, SSE, or a queue (BullMQ / Vercel Queue).
- Rationale: Vercel function cold starts + DB connection pool
  warm-up = a sub-minute cron spends most of its budget on
  bookkeeping.

### R3. Cadence ceiling: pick the slowest schedule that meets the SLA

If the freshness requirement is "within 1 hour" → run hourly,
not every 15 min. Burning function-invocation budget on
unnecessary runs ≠ "more reliable".

| Use case | Recommended cadence |
|---|---|
| News pipeline (3-5/day) | `*/30 * * * *` |
| Broker verify retry | `*/15 * * * *` |
| Office sweep / cache prune | `0 * * * *` (hourly) |
| Daily aggregate / rollup | `30 0 * * *` (00:30 UTC) |
| Retention prune | `23 3 * * *` (off-peak UTC) |
| Weekly digest | `0 2 * * 1` (Monday 09:00 BKK) |

### R4. Off-peak UTC for batch work

Hourly aggregations and prunes run **02:00 – 04:00 UTC**
(09:00 – 11:00 BKK = lowest user concurrency for our
SEA-skewed audience). Avoid `0 0 * * *` (midnight UTC = peak
traffic for many regions).

### R5. Every cron registers + logs

Every cron route MUST:
1. Have a row in `src/lib/cron/registry.ts` with `path`,
   `schedule`, `description`, `owner`, `ownerEmail`,
   `tierImpact`, `slaMs`, `estCostPerRunUsd`.
2. Call `startCronRun()` at the top of the handler.
3. Call `finishCronRun()` before every return path
   (success + failure).
4. Set `error_type` on the Sentry log per §13 (logging via Sentry).
5. Pass `cron_path: jobPath` as a Sentry tag on every `logError` /
   `logWarn` call so alert rule 564949 can filter `tag:source:background`
   AND `tag:cron_path:*`.
6. Auth gate: `x-vercel-cron: 1` header OR
   `Authorization: Bearer ${CRON_SECRET}`.

### R5a. Schedule stagger — no two crons share a minute

- Two crons firing on the same minute (e.g. `30 0 * * *` × 2) compete
  for cold-start function slots + Supabase connection pool. Caused
  contention on 2026-04-28 (`quotes-refresh` + `activity-rollup`).
- **Hard rule**: before adding a new entry to `vercel.json` `crons[]`,
  grep the file for collisions:
  ```bash
  cd "<your folder>"
  python3 -c 'import json; d=json.load(open("vercel.json")); m=[c["schedule"] for c in d["crons"]]; print("DUP" if len(m)!=len(set(m)) else "OK")'
  ```
- If `DUP`, pick a different minute. Stagger pattern (UTC):
  - **Daily aggregations** (00:30 ± 5 min): one per minute slot
  - **Daily prunes** (03:00 – 04:00 off-peak): one per minute slot
  - **Hourly sweeps**: pick a minute offset (`5`, `15`, `25`, `35`, `45`, `55` — avoid `0` if more than one hourly cron)
  - **Sub-hourly** (`*/5`, `*/15`, `*/30`): coexist OK (different intervals collide rarely)
- `/admin/cron` drift detector flags it. CI lint: `scripts/lint-cron-collision.mjs` (shipped 2026-05-17, commit f664d30) — runs as part of `.github/workflows/lint-rules.yml` + `pnpm lint:cron-collision`. PRs with duplicate schedule strings fail CI + can't merge.
- Limitation: lint catches identical-string collision only. Overlapping but non-identical schedules (e.g. `0 * * * *` hourly + `0 */6 * * *` 6-hourly both fire at 00:00, 06:00, 12:00, 18:00) are NOT caught — operator judgement. Stagger if both crons are heavy.

### R6. Cost budget per cron

- Soft cap: **$1/day per cron** (`estCostPerRunUsd × runs/day ≤ $1`).
- Hard cap (platform-wide): **$20/month** for all crons combined.
- Above $1/day → CTO approval. Above $20/month → architecture
  review (move to queue with backpressure, downsample, or kill).

### R7. SLA

Every cron declares a `slaMs` — max acceptable wall clock per
run. The `/admin/cron` dashboard turns the latency cell red on
breach. Two breaches in 24h = P2 alert (email). Five in 24h =
P1 alert (LINE/Slack).

### R8. Drift detection

The `/admin/cron` page diffs `vercel.json` against
`registry.ts`. Drift surfaces as a red badge:
- "registry-only" → scheduled in code but missing from
  `vercel.json` (won't fire). DevOps fixes.
- "vercel-only" → fires on Vercel but no registry entry =
  no metadata, no logging. Owner fixes.

## Tier limits — fan-out caps for user-facing crons

A cron that emails / pushes / pings every user must respect
the user's tier and consent. Quotas below are hard ceilings,
not targets.

### Free tier

A new free user can receive **AT MOST**:

- **3 emails in the first 7 days** (welcome series only).
  After that: zero unsolicited email until they convert or opt
  in.
- **0 push notifications** (PWA push is opt-in; default off).
- **0 streak nudges** (Free tier doesn't get streak break
  emails).
- **0 weekly digests** (Pro feature only).
- **System emails** (password reset, payment receipt, security
  alerts) are exempt from the cap.

If a feature needs more touches with a free user, the answer
is "show in-app, don't push outbound".

### Pro tier

A Pro user can receive:

- **1 weekly digest** (Monday 02:00 UTC), respects opt-out.
- **Up to 5 alerts/day** (signal hits, position breaches,
  margin-call warnings — feature-specific caps tighter).
- **Streak nudges** when streak ≥ 7 days breaks.
- **Course completion** + **certificate** emails.
- Push notifications (opt-in via PWA install prompt).

Aggregate per Pro user: ≤ **2 outbound touches/day**, ≤
**10/week**. Above this = annoying, churn risk.

### Pro tier — entitlement matrix

The Pro subscription unlocks the following surfaces. This is
the canonical list — UI gates, paywalls, and email content all
read from here. Adding a feature to Pro = update this table
first.

| Surface | Free | Pro |
|---|---|---|
| `/news`, `/guides`, `/articles` (read) | yes | yes |
| `/dashboard/backtest` (Beta — open during beta) | yes (5 cr/run) | yes (5 cr/run, priority) |
| Position size calculator | yes | yes |
| Welcome credits (100) | yes | yes |
| Subscribe to **signals** | — | yes |
| Push notification on signal hit | — | yes |
| Trade journal (manual + auto-import) | — | yes |
| Live position tracker (`/dashboard/positions`) | — | yes |
| Weekly digest email | — | yes |
| Streak nudge emails | — | yes |
| Course bundles + cohort booking | — | yes |
| 1-on-1 mentorship slots | — | by bundle |
| Backtest Lite history (more than last 10) | — | yes (last 100) |
| Custom strategy presets (save + share) | — | yes |
| Priority API quota for `/api/quant/backtest` | — | yes |
| Concurrent backtests | 1 | 3 |
| OG share image for backtest results | — | yes |

Pricing + cadence: TBD (CMO scope). Pro tier slug already lives
in `webapp_credits_subscriptions` / `webapp_credits_subscription_plans`.

### Implementation hooks

- Email send wrapper (`src/lib/email/send.ts`) reads
  `webapp_email_preferences` per user. Caps enforced before
  the SMTP call.
- Push send wrapper (`src/lib/notifications/push.ts`) reads
  `webapp_push_subscriptions` + checks daily count in
  `webapp_notification_log`.
- Crons that fan out (e.g. `/api/cron/weekly-digest`) MUST
  filter `webapp_credits_subscriptions.status='active' AND tier='pro'`
  in the SQL — never paginate every user and check tier per row.

## Anti-patterns (caught + fixed before)

- **Fire-and-forget loop in a cron handler** that takes 10
  min and times out at Vercel's 5-min cap → switch to chunking
  (process N rows/run, page state in DB).
- **`for (user of allUsers) { await sendEmail(user); }`** — 50k
  users × 200ms = 2.7 hrs. Use a queue or batch endpoint.
- **Cron that calls another cron** synchronously → fan in via
  a shared library function instead.
- **Cron that polls Vercel deploys every minute** — use the
  outgoing webhook (already wired). Cron is the backfill, not
  the primary feed.
- **Hidden setInterval in a route handler** — Vercel
  serverless instances die after 90s. setInterval = wasted
  work + occasional ghost runs.

## Operational rituals

### Weekly (CTO)

- `/admin/cron` review: any red SLA breaches? Any drift?
- 24h failure count: anything > 0 → root-cause within the day.
- Cost row: total daily $ < $1 platform-wide?

### Monthly (CTO)

- Vercel function-invocation usage trending? Compare to
  IRON-RULES §16 budget.
- Dead crons: anything in registry but `enabled: false` for
  > 30 days → delete the row + the route.

### Per-PR (Senior Dev)

When adding a new cron:
- [ ] `vercel.json` entry
- [ ] `registry.ts` entry
- [ ] `startCronRun()` / `finishCronRun()` wired
- [ ] Sentry `error_type` tag
- [ ] SLA + cost estimate filled in
- [ ] Tier impact checked (`platform` / `all-users` / `pro-only`)

## Migration checklist (existing crons → registry)

When migrating an old cron to the new pattern:

1. Add row to `CRON_REGISTRY` with best-guess metadata.
2. Wrap handler body with `startCronRun()` /
   `finishCronRun()` per the news-pipeline reference impl.
3. Smoke-test manually with `Authorization: Bearer ${CRON_SECRET}`.
4. Verify the row appears in `/admin/cron` after the next fire.
5. Remove any `console.log` style "cron fired" lines — the log
   table replaces them.

## See also

- `IRON-RULES.md` §17 (this section, canonical)
- `playbooks/news-automation.md` — example of a registered cron
- `src/lib/cron/registry.ts` — runtime source of truth
- `src/lib/cron/record.ts` — start/finish helpers
- `/admin/cron` — operational dashboard
