---
title: MoonieX Org Map — Product/Repo Overview
tags: [org-map, overview, cmo]
owner: cmo
status: living-doc, on-demand refresh (no cron)
last_refreshed: 2026-08-03
---

# Org Map

Source-of-truth data for the CEO-facing "org overview" Artifacts (mindmap +
roadmap + business model + brainstorm), one page per GitHub repo **prefix**
(brand), not per folder name. Refreshed on request — say "update the org
map" to a CMO/CTO session, not automated.

**All 4 prefixes live (2026-08-03):**
- [MoonieX](https://claude.ai/code/artifact/17b32fe9-7fb7-4f18-b893-c5a0b1eadadc) — flagship, 12+ repos
- [WarpClip](https://claude.ai/code/artifact/88a178a3-bb32-4412-9e39-6bb1f3cb0e69) — parked
- [LungNote](https://claude.ai/code/artifact/cb28140b-c948-4a64-990e-0aecb0dd528d) — pilot, business idea done
- [Chatudo](https://claude.ai/code/artifact/9a1d41bb-cc33-47e1-81ff-5d499b8b0dc6) (formerly LuNar Agent) — parked

## Prefix: MoonieX

Flagship, live at www.mooniex.com. 12+ repos grouped into 6 real clusters:
Flagship (webapp), Backend (claudeflow), Org Infra (agents, claudesign,
scriptable, claude-skills, website-templete), Trading Archive (moonx,
option — both dormant/retired), Legacy/Parked (video-engine CLOSED
2026-06-23, wa-system, genui — all dormant), Reference-only (company,
nohuman, controller-system, alphatrader — 3rd-party clones, not ours).

Real activity right now (from 71+ open issues across webapp/claudeflow/
agents): prod DDL blocked by permission classifier (claudeflow #184, fresh
today), Vercel outage caused CFO billing-rollup gaps (webapp #94), Workflow
Monitor SLA+results build-out (webapp #79/#80), GEO off-site P3-P5 (webapp
#59), LuNar separation ClaudeFlow-side (claudeflow #165, same CEO Phase-0
block as Chatudo), Console mobile live-verify (agents #37).

Business model (operational facts, not a full JTBD workup yet — flagged as
a gap, MoonieX deserves its own dedicated strategy session): IB rebate
revenue (XM ~$15/lot, Exness ~$8/lot gold-based, both 80% share — locked
numbers, never recompute from DB max), value prop = get cashback on trades
you're already making + free trading content/tools, known CGO measurement
gaps (G1-G4, broker events never fire / outbound unmeasured / Exness TikTok
attribution / Meta CAPI), regulatory risk from DSI's "Shutdown the
Laundering" crackdown (avoid HFM/QRS/GOFX-style positioning).

## Prefix: WarpClip

2 repos (WarpClip-webapp, WarpClip-design), both **PARKED 2026-05-28 — CEO
pivot to MoonieX**, confirmed in the real GitHub issue bodies (#29, #30).
11 open issues total (7 webapp — mostly stalled infra/ops: FB Pixel/BM,
LINE OA welcome msg, GSC/Bing submission, hi@warpclip.com inbox; 4 design —
P0-P3 backlog, not marked parked, just idle).

Real unresolved brand conflict (issue #29): live shipped code = dark
zinc-950 + lime + Geist + W-stroke tile; committed Brand-Brief v3.0 = light
+ B&W + Geist/Newsreader + marker sweep, never shipped; wiki ADR-0006 =
light + B&W, aspirational only. Three sources disagree, still unresolved.

Business idea: short-form video editing service, ฿500/clip draft pricing,
24h turnaround. Real competitors: CapCut, Opus Clip (free/cheap AI
auto-edit), Fiverr freelancers, TikTok's built-in editor. Service-heavy,
doesn't scale like software — matches why CEO already pivoted away, same
conclusion an external Gemini analysis reached independently on 2026-08-03.

## Prefix: Chatudo (formerly LuNar Agent)

1 repo (`mooniex-lunar-agent`), **PARKED — do not start (CEO directive
2026-07-31)**. 0% code, docs-only, 3 contradicting design revisions. Real
open bug: `claudeflow_lunar_turns` table missing in prod since 2026-06-01,
100% persist failure, eval blind for over a month (issue #2).

Separation Plan v2 (epic #9, 6 phases: #3 Freeze boundaries → #4 Extract
code → #5 Separate runtime → #6 Separate DB (8 tables) → #7 Cron→API →
#8 Cutover) — blocked at Phase 0, CEO hasn't given go-ahead.

Business idea (rev4, 2026-08-01): sell LuNar's predict→confirm admin-AI
directly to external SMC clients, not just as an internal MoonieX feature.
**Scary question, unresolved**: the same predict→confirm tech is *already
partially shipped inside MoonieX itself* (LuNar Assistant BETA, merged to
claudeflow main 2026-07-21) — so why does it need to become a separate
parked product instead of just being sold as a MoonieX broker-support
feature to MoonieX's existing customer base? This tension is the real
reason behind the 3x design churn.

## Prefix: LungNote

Personal task/note LINE-bot assistant for Thai students — "proactive
personal assistant" (todo/notes via LINE + AI agent), moving from passive
(user types) to proactive (auto-extract deadlines from Gmail).

3 repos (per LungNote's own ADR-0002, split by concern):

| Repo | Role | Status | Last commit (main) | Open issues |
|---|---|---|---|---|
| [LungNote-webapp](https://github.com/PASAKON/LungNote-webapp) | Next.js 16 app + LINE bot + AI agent, deploy lungnote.com | 🟡 dormant ~7wk (last 2026-06-15) | `d589036` 2026-06-15 | 10 open |
| [LungNote-wikis](https://github.com/PASAKON/LungNote-wikis) | Obsidian docs vault (ADRs, architecture, workflows) | 🟡 dormant on `main` since 2026-05-21 | `e6cb7b0` 2026-05-21 | 0 open |
| [LungNote-design](https://github.com/PASAKON/LungNote-design) | HTML mockups, rich menu assets | 🟡 dormant (2026-05-11) | `f845439` 2026-05-11 | 0 open |

### Stack (from LungNote-wikis/10-Architecture/Overview.md)
Next.js 16 + React 19, TypeScript strict, Tailwind 4, Supabase (Auth+DB+RLS),
Vercel. LINE OA bot → 11-tool AI agent, default model `google/gemini-2.5-flash`
(ADR-0014) with heuristic intent router escalating to Gemini 2.5 Pro
(ADR-0016). 22 ADRs total (0001–0022) — stack, auth, i18n, PWA, LINE OA,
Gmail readonly v1, memory model, quick-action chips.

### Business Model / JTBD
Full analysis: [LungNote-Wikis `30-Domain/JTBD-Positioning.md`](https://github.com/PASAKON/LungNote-Wikis/pull/19/files) (pending merge, PR #19).

**Job**: เมื่อครู/มหาลัยส่ง deadline ผ่านอีเมล (ช่องที่นักศึกษาไทยไม่ค่อยเปิด),
อยากมีอะไรเตือนผ่าน LINE เอง โดยไม่ต้องเปิดแอปใหม่ — เพื่อไม่พลาดส่งงานและไม่รู้สึกว่า
ตัวเองขี้ลืม.

**Non-obvious competitors**: ไม่ทำอะไรเลย (ใหญ่สุด), planner กระดาษ, กลุ่มไลน์เพื่อน
เตือนกันเอง, Google Calendar/Gmail reminder ฟรี, LMS มหาลัย, ถาม ChatGPT เป็นครั้งๆ,
Notion/Todoist.

**Scary question**: "ถ้า Gmail มี auto-reminder ฟรีอยู่แล้ว ทำไมต้องจ่าย ฿79/เดือน?"
— คำตอบที่มี (LINE-native, เข้าใจอีเมลไทยแบบไม่เป็นทางการ) ยังเป็น assumption
ไม่ได้ validate ด้วย customer interview.

**Business model summary**: Free (30 todo)/Pro ฿79mo·฿790yr·฿2,490 lifetime/
Education ฿39mo (spec only, issue #69 — not live). Cost driver ~$0.0013/AI-turn.
Key risk: LINE platform = single point of failure. ⚠️ Glossary's plan-tier
description และ issue #69 ไม่ตรงกัน — ต้อง sync ก่อน build จริง.

**JTBD score: 6/10** — มี job+differentiation ที่ถูกทางอยู่แล้ว แต่ไม่เคยเขียนเป็น
job statement ทางการ และยังไม่มี Mom Test interview ยืนยันเลยสักครั้ง — ราคาใน
#69 ตั้งจาก "similar SaaS" ไม่ใช่จาก job-value จริง.

### Open issues on LungNote-webapp (10, real, linked)
| # | Title | Note |
|---|---|---|
| [#80](https://github.com/PASAKON/LungNote-webapp/issues/80) | K2.6 A/B trial eval blocked | Needs CTO (Vercel creds) to run `pull-env.sh` + smoke test |
| [#79](https://github.com/PASAKON/LungNote-webapp/issues/79) | Gmail MCP Error (CEO-reported) | Screenshot filed by CEO |
| [#78](https://github.com/PASAKON/LungNote-webapp/issues/78) | Claude doesn't tag like Gmail | Small UX ask |
| [#77](https://github.com/PASAKON/LungNote-webapp/issues/77) | **Real prod bug**: shadow email-provider user duplicates real LINE-login account | Two `auth.users` rows same person; workaround applied via lungnote-mcp, root fix not shipped |
| [#76](https://github.com/PASAKON/LungNote-webapp/issues/76) | 💡 Remote MCP connector — let any user connect Claude to their LungNote | Phase 2/3 idea, product differentiator ("จดโน้ตผ่าน Claude") |
| [#75](https://github.com/PASAKON/LungNote-webapp/issues/75) | [MARKETING] Brand-Truth + voice guide | 🅿️ parked, no campaign yet |
| [#74](https://github.com/PASAKON/LungNote-webapp/issues/74) | fal.ai key decision pending | Awaiting CEO confirm — no media-gen feature yet |
| [#70](https://github.com/PASAKON/LungNote-webapp/issues/70) | **Epic**: LungNote as proactive assistant | Gmail + Subscription tracks, ~17-22 dev-days total |
| [#69](https://github.com/PASAKON/LungNote-webapp/issues/69) | Subscription system spec | Free (30 todo cap) / Pro ฿79mo, ฿790yr, ฿2,490 lifetime; LINE Pay primary + Omise PromptPay fallback |
| [#68](https://github.com/PASAKON/LungNote-webapp/issues/68) | Gmail integration spec | OAuth read + email→todo extraction, MCP-wrapped |

### Roadmap (from epic #70, real, not yet started as of last commit)
1. A1 Gmail OAuth + read (3d) → 2. S1 Tier model+quota (2d) → 3. S2 LINE Pay (3d)
→ 4. A2 Email→Todo (2d, "killer Pro feature") → 5. S3 Pro gates (2d) →
6. S4 Education discount (1d) → 7. S5 PromptPay fallback (2d) →
8. S6 Admin revenue dashboard (1d) → 9. S7 Cancel/downgrade (1d) →
10. A3 Gmail MCP wrap (2d, optional) → 11. LungNote-as-MCP-server (3d, geek tier)

### Real bug currently open
`#77` — production auth bug, same person has two `auth.users` rows (shadow
email-provider + real LINE-login). Workaround live in `mcp/lungnote-mcp`
(pinned `LUNGNOTE_USER_ID`); root identity-merge fix not shipped.
