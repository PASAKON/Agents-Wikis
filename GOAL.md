# GOAL — MoonieX North Star

**Set:** 2026-06-13 · **Owner:** CEO + CTO · **Status:** v1.1 · **Horizon:** this month (→ 2026-06-30), reviewed monthly.

This is the single primary success measure for MoonieX. It sits **above** the
daily-MVP engine (`playbooks/daily-mvp-engine.md`): shipping an MVP only counts
if it moves a metric below. Output without attribution is no longer progress.

## Vision

> **Every Thai trader trades at the lowest real cost — powered by an AI company that runs itself.**

End-state: traders win on cost (rebates + smarter tools); the company that
serves them is automated end to end. Two things must always be true — the
trader saves money, and the machine needs no human to run day to day.

## Why we rewrote this (2026-06-13)

3 months of building, ~0 measurable attributable revenue. Root cause: we
optimised **output** (tools shipped, posts made, tasks merged) instead of
**outcome** (signups, active traders), and never wired the measurement loop —
so we cannot say which post produced which baht. Building was never the test.
The test never ran. This month, it runs.

## Goals — this month

### CEO goals (the targets)

1. **Active traders (Exness + XM) > last month by ≥10%**, then sustain +5–10%
   month over month. _Active = traded ≥1 lot in the period, not just registered._
   `[baseline: pull last month's active count from Exness + XM IB portals — blocks measurement, get first]`
2. **≥1 new registration / new account per day** (hard floor, from today).
   Hit this consistently for the full window before we raise the bar.
3. **Low cost, high profit — fully automated.** Claude Code subscription is for
   **build / goal / fix only**. All runtime (content, posting, LuNar, nurture)
   runs on cron, end to end, with **no human typing "go post this,"** using API
   credits as sparingly as possible.

### CTO additions (what makes 1–3 actually achievable)

4. **Attribution is non-negotiable — the day-1 build.** Every signup and active
   trader traced to its source. See "Attribution — as-built" below for the exact
   gap and fix. Goals 1–2 are unmeasurable without this.
5. **One acquisition funnel: TikTok → `/links` → LINE OA (@mooniex / SendPulse).**
   This is the only funnel with a pulse. Facebook (2 pages) and IG are **parked**
   (0 inbound). Run it as cron: generate → approve → post, UTM on every link.
6. **Funnel targets, not just the final number.** Track the whole chain with
   conversion rates so we fix the leak, not the symptom. Reality check: 1
   signup/day cold realistically needs ~3–10 broker-link clicks → ~30–100 tool
   sessions → ~1k–5k impressions **per day**. Goal 2 is a reach problem as much
   as a copy problem — size the channel accordingly.
7. **Activate, don't just acquire.** Goal 1 is ACTIVE traders (= lots traded =
   rebate), not accounts. Build a cron re-engagement loop for dormant referred
   traders. Acquisition without activation earns 0 rebate.
8. **Automation must be observable.** No human posts → every cron self-reports
   success/failure to Gmail/LINE, and verifies the post actually landed (we have
   history of silent FB false-FAILED + posts that never went). Silent breakage =
   a dead month nobody notices.
9. **Unit economics as a number.** Set a hard monthly API-credit ceiling and a
   target cost-per-signup, so "low cost, high profit" is measurable.
   `[CEO to set: API cap ฿___/mo · target cost/signup < ฿___]`
10. **Weekly review + kill/scale ritual.** Every Sunday the CEO reads the funnel
    dashboard (5 min): scale what converts, kill what doesn't. Month-end
    (2026-06-30) = go/no-go on the whole bet, decided by DATA not mood.

## Attribution — as-built (audit 2026-06-13)

Live funnel: **TikTok → `/links` or `/links/pasakon` → broker guide → affiliate
signup**, plus **TikTok → LINE OA (`lin.ee/il21SV9`, SendPulse)**.

**Works today:** webapp internal links from bio carry
`utm_source=linkinbio&utm_medium=tiktok&utm_campaign={mooniex|pasakon}` and fire
`link_in_bio_clicked`. The TikTok → webapp hop is tracked.

**The break (money event is invisible to source):** the broker signup is the
money event, but the guide pages send the **web-default affiliate sub-IDs**, so a
TikTok-driven signup is indistinguishable from web/LINE in the broker IB portal.
Per-channel sub-IDs already exist but are unused (`link-in-bio.ts` L96–101):
XM `c=849611` (TikTok) vs `c=769658` (web); FISG `link_id=en9frakb` (TikTok) vs
`vyju7k40` (web). Exness per-channel param: TBD.

**Smallest fix (Goal 4 day-1 build):** on `/brokers/<slug>/guide`, map incoming
UTM → stamp the matching affiliate sub-ID on the signup button. Then the broker
IB portal itself reports signups / active / rebate **by source** = Goals 1 & 2
measured at the money event.

**Open dependencies:**
- Reading broker IB portal data (XM / Exness / FISG) for baseline + ongoing
  counts — likely no clean API → manual export or portal automation. Same source
  feeds the Goal 1 baseline.
- LINE OA path (`lin.ee` can't carry UTM) needs SendPulse entry-source tagging —
  design separately.

## Operating principle — build cost vs runtime cost (read before downgrading)

These are two separate budgets; do not conflate them:

- **Claude Code subscription** = BUILD capacity (CTO + DEV agents). It does NOT
  pay for runtime automation.
- **Runtime automation** (posting, LuNar, content gen) = **API credits** in
  claudeflow. This is the budget to minimise (goal 3 + goal 9).

On the plan move ($200 → $20): the $20 Pro plan **does** include Claude Code
(confirmed June 2026), but it is light — one shared 5-hour session quota + weekly
caps across chat and Code. This org spawns CTO + multiple DEV agents in parallel,
which is exactly the heavy/parallel load the $200 Max 20x tier exists for.

**Recommended sequence:** build the automation + attribution NOW on $200 (fast,
parallel), stabilise it, THEN downgrade to **Max 5x $100** (or $20 Pro) for
maintenance mode. Downgrading first slows the build, which delays the very
automation that lets us run cheap — self-sabotage. Build → stabilise → downgrade.

## Definition of success

Not "we built X." Success is one sentence with real numbers:

> "This channel → these N tool users → these M broker-link clicks → these K
> signups → +__% active traders → ฿___ rebate."

When we can write that line truthfully by 2026-06-30, MoonieX has a pulse and we
know where to push. Until we can, nothing else counts.
