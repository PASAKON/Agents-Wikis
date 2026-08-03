# Playbook — Workflow balance precalculate (the rule)

> **Hard-rule citation**: [IRON-RULES §28](https://github.com/PASAKON/MoonieX-Wikis/blob/main/IRON-RULES.md#section-28--workflow-balance-precalculate-ceo-directive-2026-05-18) (`mooniex:IRON-RULES.md`) — CEO directive 2026-05-18.
> **Owner**: pipeline maintainers — every project that burns a paid API.
> **Status**: live, mandatory for every paid-API workflow.

**Never start a workflow that spends money without pre-checking the balance.**
This page carries the rule, the estimation formula, the per-provider balance
query, and the failure modes — all provider-generic.

**The claudeflow implementation** (module skeleton, wiring, usage from a
workflow) is MoonieX code and lives in
[MoonieX's copy](https://github.com/PASAKON/MoonieX-Wikis/blob/main/playbooks/balance-precheck.md)
(`mooniex:playbooks/balance-precheck.md`). Split 2026-08-03 per ADR 0013.

## Why

A workflow that runs out of credit mid-batch leaves orphan intermediate state — Drive folders with `_MANIFEST.json` stuck at `stage='tts'` and no output, Supabase blobs that nothing references, partial Telegram alerts. The 30% buffer catches the near-zero case **before** the first paid submit and surfaces a clean BLOCK + Telegram alert instead of a half-finished run.

Real precedent: during the AXI lipsync session 2026-05-03 the fal.ai account hit `$5` floor mid-batch and Replicate rate-limited new submits to 6/min with "less than $5.0 in credit" — caught at submit, but only after the work was already in flight. §28 prevents that race upstream.

## When to invoke

- Before the **first paid API call** of every workflow entry point.
- Once per workflow run, not per call (cache the balance ≤ 60s).
- Even on small workflows ($0.50). No minimum.

## Estimation formula

### Single-model workflow

```
estimated_workflow_cost = N_parallel_calls × per_call_cost
```

Per-call cost is computed from §28.2's cost table.

### Chained workflow (e.g. lipsync → upscale)

```
estimated_workflow_cost = Σ step_i.estimated_cost
```

Each step computed independently.

### Example: MYPASAKON 3-chunk lipsync v3

```
audio chunks   = 3 × 20s
fal v3 rate    = $0.16/sec
per_call_cost  = 20 × $0.16 = $3.20

estimated_cost = 3 × $3.20 = $9.60
required (×1.30) = $9.60 × 1.30 = $12.48

decision:
  balance >= $12.48        → proceed
  $9.60 <= balance < $12.48 → WARN + proceed
  balance < $9.60           → BLOCK with deficit
```

### Example: MYPASAKON 1-chunk lipsync v3 + Topaz upscale (chained)

```
step 1 lipsync v3:    20 × $0.16 = $3.20
step 2 Topaz upscale: 20 × $0.08 = $1.60   (4K tier)
                      ─────────────────────
estimated_cost                 = $4.80
required (×1.30)                = $6.24
```

## Balance query

### fal.ai

```js
const res = await fetch('https://fal.ai/api/billing/balance', {
  headers: { Authorization: `Key ${process.env.FAL_API_KEY}` },
});
const { balance_usd } = await res.json();
```

### Replicate

Replicate does not expose a direct `/balance` endpoint. Two options:

1. Make a dummy GET against `/v1/account` and inspect the response headers for rate-limit hints (`"less than $X.0 in credit"` message under throttle).
2. Track spend locally — sum `cost_usd` from recent predictions via `/v1/predictions` listing.

Use option 1 in the BLOCK threshold check; track local spend for trend analytics.

### Other providers

OpenAI, Anthropic, ElevenLabs, Deepgram — each exposes a `/usage` or `/billing` endpoint. Wire when the workflow first uses them. Until wired, log "balance check unsupported" and proceed (WARN-tier — do not BLOCK on unknown).

## Failure modes

| Failure | Behaviour |
|---|---|
| `getProviderBalance` 5xx / network fail | Log error, treat as WARN-tier event (notify CEO), proceed. Balance unknown is not deficit. |
| `notifyLowBalance` Telegram fail | Swallow + log. Do not block the workflow on alerting failure. |
| BLOCK fires on legitimate top-up race | Re-run after CEO confirms top up via Telegram. Do not lower 1.30x buffer. |
| Multiple WARN within 60s | Cache (`_balanceCache`) means single fetch; only one alert per workflow run. |
| Provider cost rate changes | Update `IRON-RULES §28.2` cost table FIRST, then this playbook, then the helper module. Order of authority per §28.7. |

## Anti-patterns

- Bad: `try { await fetch(fal_balance) } catch {}` and proceed — silent BLOCK skip.
- Bad: Hard-coded `BALANCE_THRESHOLD = 10` constant outside the rule. Use §28.1's deterministic formula.
- Bad: Buffer < 1.30x because "the workflow always finishes under budget" — anecdote, not data; the buffer covers rate variance + retry cost.
- Bad: Submitting one fal call, checking balance, submitting next — race condition + rate-limit fan-out. One precheck per workflow run.
- Bad: "I'll add precheck after MVP" — the failure mode is mid-batch orphan state; MVP without precheck has the bug from day one.

## Cost + SLA

- Balance query: ~200ms per workflow start. Negligible.
- Telegram notify: ~500ms when WARN fires. Async; doesn't block submit.
- BLOCK saves the run cost; WARN proceeds at zero workflow-time penalty.

## Related

- [IRON-RULES §28](https://github.com/PASAKON/MoonieX-Wikis/blob/main/IRON-RULES.md#section-28--workflow-balance-precalculate-ceo-directive-2026-05-18) (`mooniex:IRON-RULES.md`) — canonical rule
- [IRON-RULES §27](https://github.com/PASAKON/MoonieX-Wikis/blob/main/IRON-RULES.md#section-27--drive-folder-convention-ceo-directive-2026-05-18) (`mooniex:IRON-RULES.md`) — Drive folder convention (related, not coupled)
- [`fal-queue-jobs.md`](fal-queue-jobs.md) — fal.ai queue contract (webapp side)
- [`projects/mooniex-claudeflow.md`](../projects/mooniex-claudeflow.md) — claudeflow project facts

## Verification

1. Add a workflow that calls `precheckWorkflow()` before submit.
2. Temporarily set fal balance below estimated_cost (top-up trick: change `FAL_API_KEY` to a low-credit test key).
3. Run workflow → should throw `BalanceBlockError` before first fal call lands.
4. Restore key + top-up → balance > estimated_cost x 1.30 → run proceeds, no Telegram.
5. Set balance between estimated_cost and x1.30 → Telegram alert fires once, workflow proceeds.

If any of these don't behave per spec, the helper module has a bug; do not loosen the rule.
