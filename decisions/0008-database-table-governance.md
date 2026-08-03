# ADR 0008 — Database table governance (anti-sprawl)

- **Status:** Accepted
- **Date:** 2026-06-01
- **Author:** CTO (per CEO directive — "Supabase tables เยอะมาก, มีกฏไหม")
- **Supersedes / relates:** extends [IRON-RULES §4](https://github.com/PASAKON/MoonieX-Wikis/blob/main/IRON-RULES.md#section-4--database--supabase) (`mooniex:IRON-RULES.md`) (prefix convention) and [§12](https://github.com/PASAKON/MoonieX-Wikis/blob/main/IRON-RULES.md#section-12--database-migrations-mooniex-webapp) (`mooniex:IRON-RULES.md`) (migration runner). New canonical rule: [IRON-RULES §30](https://github.com/PASAKON/MoonieX-Wikis/blob/main/IRON-RULES.md) (`mooniex:IRON-RULES.md`).

## Context

Shared Supabase project `tlokhyqpthvxabweekps`. Audit on 2026-06-01 (CTO):

| Repo | Tables | Prefix | Schema | Migration hygiene |
|------|-------:|--------|--------|-------------------|
| `mooniex-webapp` | 70 | `webapp_*` | `public` | Clean — 66 append-only timestamped migrations, single `__webapp_init` foundation |
| `mooniex-claudeflow` | 16 | `claudeflow_*` | `public` | **Fragmented** — timestamped migrations AND loose `scripts/*.sql` ad-hoc DDL (`create-admins-table.sql`, `migration-agents-phase1.sql`, `create-mr-golf-escalations.sql`, …) with no checksum/tracking |
| **Total** | **86** | — | all `public` | — |

86 tables across two products is not inherently excessive. The real defect is **claudeflow lets tables be born outside the migration pipeline** — ad-hoc `scripts/*.sql` run by hand on Supabase. No source-of-truth schema, no checksum, no review gate. That is the sprawl vector: when an AI/DEV can `CREATE TABLE` anywhere, every new feature grows a new table instead of reusing one.

Root cause is **governance, not count**: no rule said "extend before create", and no single schema file existed for an AI to read before emitting DDL — so models invent tables they think should exist.

## Decision — six hard rules

1. **Migration-only DDL.** No `CREATE TABLE` / `ALTER TABLE` outside `supabase/migrations/<ts>__<prefix>_<descriptor>.sql`. `scripts/*.sql` ad-hoc DDL is banned. (webapp already enforces via the §12 runner; claudeflow must adopt the same.)
2. **Extend before create.** Before adding a table, the DEV must show in the PR/task report that no existing table + column (or a `jsonb` field) can hold the data. Default to adding a column, not a table. New table requires a one-line justification.
3. **CTO / issue gate on new tables.** A migration that adds a *new* table needs an approving task or GH issue. Column adds and index changes don't. Keeps a human/CTO checkpoint on schema growth without slowing routine ALTERs.
4. **Schema snapshot = source of truth.** Each repo keeps a generated `supabase/schema.sql` (full current DDL, regenerated from migrations). AI/DEV reads this file *before* writing any migration — RAG grounding that stops hallucinated/duplicate tables.
5. **Read-only role for AI queries.** Agents that *query* the DB use a read-only Postgres role / read replica. Writes only ever land through reviewed migrations (§12). Never hand a non-deterministic process a read-write service-role key for ad-hoc SQL — prompt injection or a hallucinated `DROP` then has prod-write reach.
6. **Namespace discipline stays.** Keep the `webapp_*` / `claudeflow_*` / `option_*` prefix rule (§4); `scriptable_*` is **reserved** for `mooniex-scriptable` (added 2026-06-24 — reservation only, no tables yet; create one only via a reviewed migration following the rules above). Real Postgres schemas (`CREATE SCHEMA webapp`) are the long-term target but out of scope for this ADR — prefixes hold for now.

## Efficiency standards (use the DB well, not just sparingly)

- Index every column used in a `WHERE` / `JOIN` / `ORDER BY`; **GIN index** on any `jsonb` column that is queried.
- Use embedded/nested selects (PostgREST `select=...,rel(...)`) instead of N+1 round-trips.
- Connection pooling via Supavisor/PgBouncer for serverless/cron callers.
- Partition or time-bucket high-volume append-only tables (`webapp_*_log`, `claudeflow_agent_runs`, message tables); schedule retention pruning (webapp already has `*-prune` crons).
- Keep RLS policies cheap — they run per row; avoid subqueries in policies on hot tables.
- Prefer views for repeated complex joins; `VACUUM ANALYZE` cadence on churny tables.

## Consequences

- claudeflow gains the same migration discipline webapp has (cleanup task dispatched 2026-06-01).
- Both repos get a committed `supabase/schema.sql` for AI grounding.
- Slightly more friction to add a table (justification + gate) — intended. Adding a column stays frictionless.
- Read-only AI query role is a follow-up infra task (DevOps) — flagged, not yet built.

## How to apply (DEV checklist before any schema change)

1. Read `supabase/schema.sql` (the snapshot). Does a table already hold this?
2. Can a new **column** or `jsonb` key serve instead of a new table? If yes → do that.
3. If a new table is truly needed → write a timestamped migration in `supabase/migrations/`, prefix-correct, idempotent (`IF NOT EXISTS`), with a one-line justification + approving task/issue.
4. Never run `scripts/*.sql` by hand on Supabase. Never `CREATE TABLE` from an ad-hoc agent session.
5. Regenerate `supabase/schema.sql` and commit it with the migration.
