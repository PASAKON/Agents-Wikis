---
title: "ADR — Approval Gate (preference-learning HITL gate for AI automation)"
date: 2026-05-26
status: Accepted
owner: CTO
deciders: [CEO, CTO]
projects: [warpclip-webapp, mooniex-webapp, mooniex-claudeflow, lungnote-webapp]
---

# ADR — Approval Gate

## Status
Accepted 2026-05-26 (CEO). Pilot = **WarpClip ad creative**.
**Host re-homed to ClaudeFlow 2026-05-26 — see Amendment at end.**

## Context

AI-generated automation outputs (ClaudeFlow poster scripts, WarpClip ad
creative, news articles, etc.) currently ship through three unrelated,
half-built approval mechanisms:

- **News pipeline** (mooniex-webapp): hardcoded score-gate
  (`<0.65` reject / `0.65–0.85` human draft / `≥0.85` auto-publish). No
  feedback capture, no learning.
- **Workflow-pause email approve** (claudeflow): HMAC one-tap approve URL
  + Gmail-poll for "approve/done". A human channel, but per-incident only.
- **pgvector** already live in claudeflow (`videolib.js`) on the shared
  Supabase parent `tlokhyqpthvxabweekps`.

There is no shared, reusable way to (a) gate AI output behind a human,
(b) capture *why* a human approved/rejected, (c) improve the generator
from that feedback, and (d) eventually let the system approve on the
human's behalf. The 2026-05-26 WarpClip off-brand ad incident
(generator produced wrong-color/off-theme creative because no brand
rule was enforced) is the concrete cost of having no such gate.

## Decision

Build one reusable **Approval Gate**: a human-in-the-loop gate that
captures structured feedback, refines the generator prompt on every
reject, and **graduates per-workflow** from human approval to
vector-based auto-approval once it can prove it agrees with the human.

### Graduation ladder (per workflow `key`, independent)

1. **HUMAN** — AI generates → human approve/reject + `score` + `reason_codes`.
   Each decision is embedded and stored labelled (approved=positive,
   rejected=negative+reason). On reject → refine loop.
2. **SHADOW** — vector classifier *predicts* approve/reject; human still
   decides. Track agreement (predicted vs actual). No behaviour change yet.
3. **AUTO** — flip when shadow agreement ≥ `target_agreement` (default
   0.95) over the last K decisions AND ≥ `min_samples` (default 50).
   New artifact: kNN vs labelled exemplars →
   - `sim_approve ≥ auto_threshold` AND `sim_reject < reject_guard` → auto-approve
   - `sim_reject ≥ reject_guard` → auto-reject → refine loop
   - else (novel/uncertain) → **escalate to human** (permanent safety valve)

   Auto mode keeps an uncertain-band human escalation forever, a
   per-workflow kill switch, and auto-demotes to SHADOW on drift.

### Refine loop (on any reject)

Haiku diagnoser reads `artifact + reason_codes + reason_text + prompt`,
emits a **minimal patch** appended to the workflow's human-readable
`learned_rules[]` block (NOT a prompt rewrite). Bump `prompt_version`,
regenerate, loop to `iteration` cap 3, then escalate to human with the
attempt history. Rolling reject-rate per workflow = the "error" metric
*and* the graduation-readiness gate. `learned_rules[]` is what prevents
recurrence of the WarpClip off-brand incident.

### Image artifacts (WarpClip pilot) — two layers

- **Layer 1 — deterministic brand-gate** (no ML, ~$0): palette/hex check
  (must contain lime `#CCFF00` + zinc-950, no off-palette hues), aspect
  ratio, logo (W-on-lime tile) present, caption length/banned words.
  Hard fail → straight to refine loop, never reaches human or vector.
- **Layer 2 — multimodal embedding** for preference/style learning:
  embed rendered image + caption together via **Cohere
  `embed-multimodal-v3`** (one image+text vector space; hosted; avoids
  fal.ai per the WarpClip brand-truth rule, which governs *generation*).
  Text-only embedding rejected for a visual pilot — blind to colour/layout.

For text workflows (poster script) text embedding (`text-embedding-3-small`)
is sufficient.

## Architecture

Shared `approval_*` tables in Supabase parent `tlokhyqpthvxabweekps`
(pgvector). *(Host: see Amendment — the gate logic lives in ClaudeFlow,
not mooniex-webapp. The shared tables below are host-agnostic.)*

```
approval_workflows     key · project · mode(human|shadow|auto) · auto_threshold
                       · min_samples · target_agreement · prompt_version
                       · learned_rules[]
approval_requests      workflow_key · artifact_payload(jsonb) · artifact_text
                       · context(jsonb) · embedding(vector) · predicted_decision
                       · predicted_score · status · iteration · parent_request_id
approval_decisions     request_id · decided_by · decision · score(1–5)
                       · reason_codes[] · reason_text · tags[]
approval_prompt_patches workflow_key · from/to_version · diagnosis · patch
```

Human channels: `/admin/approvals` surface in mooniex-webapp (rich review
+ reason chips) + reuse claudeflow HMAC email-approve link (one-tap mobile).
News-pipeline score-gate migrates onto this later (follow-up, not pilot).

## Consequences

- **+** One gate replaces three half-solutions; reusable across all 4 repos.
- **+** Explainable (kNN cites the exemplars it matched), reversible, no
  retraining job, grows monotonically.
- **+** Cheap: Layer-1 = $0; Cohere multimodal ≈ $0.0001/img + Haiku
  diagnoser on rejects. Inside IRON-RULES §16 $20/mo cap.
- **−** Cold-start cost: ~50 human decisions per workflow before auto.
- **−** New Cohere dependency (multimodal embedding) — new API key + cost line.
- **Risk:** premature flip to auto → mitigated by SHADOW agreement gate +
  uncertain-band escalation + kill switch + drift auto-demote.

## Pilot build order (as built — host later moved, see Amendment)

1. Supabase migration: `approval_*` tables + pgvector index + seed
   `warpclip.ad_creative` workflow (mode=human). ✅ merged
2. Core lib `src/lib/approval/` (mooniex-webapp): requestApproval /
   recordDecision / embed / predict(kNN) / refine(Haiku). ✅ merged
3. Layer-1 WarpClip brand linter. ✅ merged
4. `/admin/approvals` review surface. ✅ merged
5. Ingest endpoint (POST → requestApproval) so external producers go
   through the linter instead of raw-inserting. ✅ merged
6. WarpClip generator → gpt-image-1 (OpenAI, no fal) + lime + private
   Supabase staging + POST to the ingest endpoint. ✅ merged (warpclip local)

SHADOW + AUTO mode logic and email-approve reuse are follow-ups after the
HUMAN-phase pilot collects decisions.

## Amendment 2026-05-26 — Host = ClaudeFlow

CEO 2026-05-26: re-home the gate to **ClaudeFlow** — the always-on
automation hub (runs almost all automation; already cross-platform, has the
email-approve infra, on the shared Supabase parent). This supersedes the
"Architecture" line "brains live in mooniex-webapp."

- **Gate logic** (requestApproval / recordDecision / embed / predict /
  refine / brand-lint) + ingest routes `POST /api/approval/requests` and
  `POST /api/approval/decisions` → **ClaudeFlow** webhook server
  (`src/approval/` + `src/webhook/server.js`), public at
  `https://webhook.srv1395225.hstgr.cloud`. Auth `x-approval-token` /
  `APPROVAL_INGEST_TOKEN`.
- **Tables unchanged** — shared in Supabase parent `tlokhyqpthvxabweekps`,
  host-agnostic. The migration already applied still stands.
- **Producers** (WarpClip, LungNote, MoonieX, ClaudeFlow's own crons) POST
  to ClaudeFlow. WarpClip generator change = env `APPROVAL_INGEST_URL` only
  (endpoint paths match, so no code change).
- **Admin review UI** stays in mooniex-webapp `/admin/approvals` (ClaudeFlow
  is headless) — reads the shared tables, posts decisions to ClaudeFlow
  `/api/approval/decisions`.
- The mooniex-webapp lib + `/api/approval/requests` route built during the
  pilot are **superseded** by the ClaudeFlow port (the UI survives).
- Migration: claudeflow port `task-158017d5`; webapp repoint (after port
  merges); warpclip env-only. Deploy to VPS on CEO request
  (`version:vps:promote`).
