# Secrets rotation — env vars + tokens

> Owned by **CTO**. Rotation cadence per secret class. Schedule reminders weekly in the CTO digest. Compromise / leak → immediate rotation, regardless of cadence.

This playbook covers the recurring rotation of every secret the project depends on. Each section names: where the secret lives, how to mint a fresh one, the swap procedure (zero-downtime), and the verification step.

**Hard rules (apply to every secret):**

1. **Never paste in chat / commits / logs / Vercel build logs.** `printf` to stdin only, never `echo` (trailing `\n` breaks string compare per IRON-RULES §2 #5).
2. **Verify last 8 chars after pull** — `vercel env pull .env.tmp && grep <NAME> .env.tmp | tail -c 8 && rm .env.tmp` to confirm the value landed without truncation.
3. **Both prod + preview every time.** A secret missing from one tier surfaces as a 401 on a deploy you didn't expect.
4. **Single source-of-truth file per machine.** Local: `~/.claude/<project>.json` (mode 0600) for hook configs. Never copy across machines without a documented hand-off.
5. **Audit trail in `LLMs/MIGRATIONS.md`** — append a "Rotated `<NAME>` on `<UTC date>`" entry every time. CTO weekly digest reads from this list.

---

## Rotation cadence (CTO digest schedule)

| Secret | Cadence | Last rotated | Owner |
|--------|---------|--------------|-------|
| `OFFICE_TOKEN` | quarterly (90 d) | 2026-04-26 | DevOps |
| `CRON_SECRET` | quarterly (90 d) | 2026-04-27 (provisioned) | DevOps |
| `SUPABASE_ACCESS_TOKEN` (PAT) | quarterly (90 d, hard) | (per IRON §12) | DevOps |
| `SUPABASE_SERVICE_ROLE_KEY` | yearly (or on suspected leak) | per Supabase rotate | DevOps |
| `EA_API_KEY` (MT5 webhook) | quarterly | (TBD) | DevOps |
| `LINE_CHANNEL_SECRET` | yearly (or on LINE provider rotate) | (TBD) | SrDev who owns LINE bridge |
| `SENTRY_API_TOKEN` (admin) + `SENTRY_API_TOKEN_RW` | quarterly (90 d) | 2026-04-27 — next 2026-07-26 (scheduled task `rotate-sentry-token-2026-05-04`) | DevOps |
| `FAL_API_KEY` | yearly (admin-scope key) | (TBD) | DevOps |
| `TWELVE_DATA_API_KEY` | yearly | (TBD) | DevOps |
| `OUTLOOK_REFRESH_TOKEN` / `GMAIL_REFRESH_TOKEN` | refresh on revoke | per OAuth | DevOps |
| `mooniex-option` `.env.enc` passphrase | yearly | per env-push.js doc | option agent |

CTO Monday morning digest reads this table + weeks-since-last-rotation column. Anything > cadence + 7-day grace = flag.

---

## Standard rotation procedure (zero-downtime)

The pattern works for **any Vercel-stored secret** (`OFFICE_TOKEN`, `CRON_SECRET`, `EA_API_KEY`, `SUPABASE_ACCESS_TOKEN`, …). It's two-phase: deploy the new value, retire the old.

### Phase 1 — provision new value alongside old

```bash
NEW=$(openssl rand -hex 32)                     # or per-secret minting
printf "%s" "$NEW" | vercel env add <NAME> production main
printf "%s" "$NEW" | vercel env add <NAME> preview main
# verify
vercel env ls | grep <NAME>                     # both rows present
```

If the secret has a downstream consumer outside Vercel (e.g. local hook config, GitHub Actions secret, mooniex-claudeflow VPS env), update those **before** the next step. The old value stays valid until you cut traffic over.

### Phase 2 — cut traffic + remove old value

```bash
# Option A — accept overlap: both old + new accepted by route handler
#   Implement OR-comparison in code: header === Bearer ${OLD} || === Bearer ${NEW}
#   Land + deploy; observe 24h; remove old value next Monday.

# Option B — atomic flip: route reads only one value (current code)
#   Trigger one fresh deploy after Phase 1 so the new value is baked.
#   Old value still active in current Vercel cache for ~5 min before
#   cold-start instances pick up the new env. Retry on 401 during this window.
```

After overlap window:

```bash
vercel env rm <NAME> production -y
vercel env rm <NAME> preview -y
```

Audit row:

```bash
echo "$(date -u +%Y-%m-%d) CTO rotated <NAME>" >> /Users/gob/projects/LLMs/MIGRATIONS.md
```

---

## CRON_SECRET — provision (first time)

Required by `/api/cron/*` handlers when called outside the Vercel Cron infrastructure (manual smoke tests, ad-hoc reruns). Optional for production cron firing — Vercel attaches `x-vercel-cron: 1` automatically on its scheduled triggers per IRON-RULES §3.

```bash
SECRET=$(openssl rand -hex 32)
cd "/Users/gob/projects/mooniex-webapp (Desk-A)"

printf "%s" "$SECRET" | vercel env add CRON_SECRET production main
printf "%s" "$SECRET" | vercel env add CRON_SECRET preview main
vercel env ls | grep CRON_SECRET                   # 2 rows expected

# Save locally for smoke tests (mode 0600, never committed)
mkdir -p ~/.claude
printf '{"cron_secret":"%s"}\n' "$SECRET" > ~/.claude/cron-smoke.json
chmod 600 ~/.claude/cron-smoke.json

unset SECRET
```

Trigger a fresh prod deploy after the env add so existing serverless instances pick up the value (Vercel does NOT hot-reload baked env vars).

### Smoke test pattern

```bash
TOK=$(node -e 'process.stdout.write(JSON.parse(require("fs").readFileSync(process.env.HOME+"/.claude/cron-smoke.json","utf8")).cron_secret)')

curl -s -X GET https://www.mooniex.com/api/cron/api-usage-prune \
  -H "Authorization: Bearer $TOK"
# expected: {"ok":true,"deleted":N,"cutoff":"...","retention_days":90}

curl -s -X GET https://www.mooniex.com/api/cron/audit-prune \
  -H "Authorization: Bearer $TOK"
# expected: {"ok":true,"deleted":N,"cutoff":"...","retention_days":365}

curl -s -X GET https://www.mooniex.com/api/cron/office-sweep \
  -H "Authorization: Bearer $TOK"
# expected: {"ok":true,"released":N,"released_ids":[...]}

unset TOK
```

`401 unauthorized` = env not propagated (need fresh deploy) or token wrong.

---

## CRON_SECRET — rotation (every 90 days)

Standard procedure (zero-downtime, Phase 1 + Phase 2 above). Cron handlers only check one value at a time per current code, so do the **atomic flip** path:

1. `vercel env rm CRON_SECRET production -y` then re-add with new value
2. `vercel env rm CRON_SECRET preview -y` then re-add
3. Trigger fresh prod deploy
4. Update `~/.claude/cron-smoke.json`
5. Smoke test: `curl … /api/cron/office-sweep` should return 200
6. `LLMs/MIGRATIONS.md`: append rotation entry

**Don't rotate during a live Vercel cron firing window.** The current implementation accepts `x-vercel-cron: 1` OR Bearer, so legitimate Vercel-triggered runs aren't affected by token changes — but spending the rotation 1 minute after a cron fires is the safest cushion.

---

## OFFICE_TOKEN — rotation

Same pattern. Local config file: `~/.claude/office.json`. Hook scripts read it at every PreToolUse / UserPromptSubmit / Stop, so a stale value blocks all desk claims silently (hooks fail-open per design).

After rotation, `node --check ~/.claude/hooks/office-pretool.mjs` then trigger a Bash command in any desk folder to verify the hook claims succeed.

---

## SUPABASE_ACCESS_TOKEN — rotation (HARD 90 days)

Per IRON-RULES §12. Three locations to update in lockstep:

1. **Vercel** — `mooniex-webapp` project env (preview + production)
2. **GitHub Action secret** — `gh secret set SUPABASE_ACCESS_TOKEN -b "sbp_..."`
3. **Local `.env.local`** for `node scripts/migrate.mjs --dry-run` development

Skipping (3) just breaks local dry-runs. Skipping (1) or (2) breaks prod migrations.

After rotation, run a no-op migration through the GH Action to verify:

```bash
gh workflow run db-migrate -f dry_run=true
gh run watch --repo PASAKON/mooniex-webapp
```

Expect: `applied: 0`, no 401.

---

## Compromise / leak procedure (any secret)

1. **Rotate now** — don't wait for cadence.
2. Append to `LLMs/MIGRATIONS.md` with `[LEAK]` tag + suspected exposure surface (commit log, chat log, Vercel build log, screenshot).
3. Audit `webapp_audit_log` for the past 24h windowed on the affected route.
4. If financial impact possible (Omise / FAL billing keys, Vercel deploy hooks): notify CEO immediately; escalate per IRON §15.
5. If leaked into a public commit: `gh secret set` first, then `git push --force-with-lease` to rewrite history (requires CEO sign-off per §2 conflict resolution).

---

## What NOT to do

- ❌ Reuse `OFFICE_TOKEN` for `CRON_SECRET` (different blast radius — token used cross-cron should be CRON_SECRET only).
- ❌ Hard-code rotation date in source — the table above is the audit trail.
- ❌ Leave both old + new env vars in Vercel after Phase 2 — clutter + ambiguity.
- ❌ Rotate `OFFICE_TOKEN` while desks are mid-claim — hook auto-recovery handles cross-day, NOT cross-token-rotation.
- ❌ Generate the new value via `cat /dev/urandom` (no hex encoding; pipe-as-string breaks). `openssl rand -hex 32` always.
