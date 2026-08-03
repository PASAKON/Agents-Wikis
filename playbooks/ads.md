---
title: "Ads Playbook (paid media operations)"
tags: [playbook, ads, paid-media, ads_manager]
status: Active
date: 2026-05-26
---

# Ads Playbook

Operational reference for the `ads_manager` worker and the CMO who briefs them:
naming convention, default budgets, campaign templates, creative spec, the hard
production rule, reporting cadence, and the tools map. Brand-agnostic — every
brand run on this org uses it.

**Live account / Page / Pixel IDs are NOT here.** Those are brand data and live
in that brand's own wiki. MoonieX's:
[Ads — MoonieX account / page / pixel map](https://github.com/PASAKON/MoonieX-Wikis/blob/main/playbooks/ads.md)
(`mooniex:playbooks/ads.md`).

> Split out of the single `playbooks/ads.md` on 2026-08-03 per ADR 0013
> (`mooniex:decisions/0013-wiki-scope-boundary.md`).

## 🔥 Brand Truth (READ FIRST before briefing creative)

Every paid asset must align with the per-brand canonical Brand-Truth doc:

| Brand | Brand-Truth canonical |
|-------|------------------------|
| WarpClip | [`WarpClip-wikis/10-Architecture/Brand-Truth.md`](https://github.com/PASAKON/WarpClip-wikis/blob/main/10-Architecture/Brand-Truth.md) |
| LungNote | TBD (write before first campaign) |
| MoonieX (parent) | `mooniex-voice-guide.md` |

**Workflow:** CMO Pre-work checklist requires Brand-Truth verification
protocol — see `roles/cmo.md` § "Brand Truth Protocol". `ads_manager`
+ `web_designer` + `content_strategist` all have the same gate.

**Rule:** NO image gen API for brand-strict creative. Manual composition only.

## Naming Convention

```
<brand>_<objective>_<audience>_<creative-id>_<YYYYMMDD>
```

Examples:
- Campaign: `WarpClip_Sales_Cold_LandingV2_20260525`
- Ad set:   `WarpClip_Sales_Cold_Interest-Reels-Editing_20260525`
- Ad:       `WarpClip_Sales_Cold_VideoA_B-Roll-Premium_20260525`

Mandatory tokens:
- brand = `WarpClip` | `MoonieX` | `LungNote` | etc.
- objective = `Awareness` | `Traffic` | `Engagement` | `Leads` |
  `Sales` | `AppInstall`
- audience = `Cold` | `LAL1` | `LAL5` | `RT-PageView` | `RT-AddToCart` |
  `Custom-LINE` | `Custom-Email`

## Default Budgets (USD)

| Stage | Daily | Lifetime | Note |
|---|---|---|---|
| Test (creative learning) | $20 | $100 (5 days) | run 3-5 ad sets, kill bottom 50% on day 3 |
| Scale (winning creative) | $50-100 | varies | only after CTR > 1% AND CPC < benchmark |
| Retargeting | $10-20 | varies | small audience, frequency cap 3 / 7 days |

Per-campaign caps before CFO sign-off:
- WarpClip: $300 lifetime (CEO-approved 2026-05-25)
- Others: $200 lifetime default

## Standard Campaign Templates

### Sales (Conversion API path)
- Objective: `OUTCOME_SALES`
- Optimization: `OFFSITE_CONVERSIONS`
- Conversion event: `Purchase` (or `Lead` for service business)
- Bid strategy: `LOWEST_COST_WITHOUT_CAP` until learning phase exits
- Placements: Advantage+ (recommended) — exclude Audience Network
  unless cost forces it
- Attribution: 7-day click + 1-day view (default)

### Sales (Traffic fallback, no Pixel)
- Objective: `OUTCOME_TRAFFIC`
- Optimization: `LANDING_PAGE_VIEWS`
- Destination: LINE OA URL (`line.me/R/ti/p/@xxx`) or landing → LINE CTA
- Bid: `LOWEST_COST_WITHOUT_CAP`
- Use UTM: `utm_source=fb&utm_medium=paid&utm_campaign=<slug>&utm_content=<creative-id>`

### Click-to-Messenger (NEW — recommended for cold paid traffic)
- Objective: `OUTCOME_ENGAGEMENT`
- Optimization: `MESSAGING_CONVERSATIONS_STARTED` (or `CONVERSATIONS`)
- Destination: Page Messenger
- No Pixel/CAPI dependency — conversation tracking native in Meta
- Requires Page linked to ad account
- CTA: "ทักผ่าน Messenger" / "ส่งข้อความ"

### Leads (LINE OA capture)
- Objective: `OUTCOME_LEADS`
- Optimization: `LEAD_GENERATION` (requires Page LeadGen ToS accepted)
- Form: native instant form OR redirect to LINE OA
- Note: Thai SMB audiences convert better on LINE redirect than
  native form. Test both.

## Creative Spec (Meta-compliant)

| Placement | Ratio | Min res | Max video |
|---|---|---|---|
| Feed | 1:1 or 4:5 | 1080x1080 | 60s |
| Reels / Stories | 9:16 | 1080x1920 | 60s |
| Right column | 1:1 | 1080x1080 | n/a |

Text rule (current Meta policy): text overlay on image is not gated
by 20% rule anymore, but excess text still hurts delivery — keep
< 30% of frame.

## Creative Production Rule (HARD)

For brand-strict assets (logo, ad image, social, landing hero):
- **NO image gen API** (fal.ai / gpt-image-2 / midjourney). These
  hallucinate hex codes + reverse contrast. Verified failure 2026-05-26
  (WarpClip ad rejected, $0.33 wasted).
- **Manual composition only:**
  - HTML/CSS → Playwright screenshot → PNG (recommended)
  - Python PIL/cairo composing existing SVG + font
  - SVG → rsvg-convert / Inkscape CLI
- **Reuse existing SVG mark** — never recreate from memory.
- **Commit composition source** (`.html` / `.py`) alongside output PNG
  for reproducibility.
- AI image gen OK for: stock photo / illustrative b-roll only, not
  brand-strict surfaces.

## Reporting Cadence

| Frequency | Owner | Output |
|---|---|---|
| Daily | `ads_manager` | spend, impressions, clicks, CTR, CPC, CPM snapshot |
| 3-day | `ads_manager` → CGO | creative-level CTR + cost-per-result, kill bottom 50% |
| 7-day | CGO | full funnel, ROAS calc, scale / kill recommendation |
| End-of-campaign | CMO + CGO | retro ADR in `decisions/` |

## Tools Map

| Task | Tool / MCP |
|---|---|
| Account / page discovery | `mcp__claude_ai_Facebook_Ads__ads_get_ad_accounts` + `ads_get_user_pages` |
| Campaign / ad set / ad CRUD | `mcp__claude_ai_Facebook_Ads__ads_create_*` |
| Custom audience | `mcp__claude_ai_Facebook_Ads__ads_create_custom_audience` |
| Industry benchmark | `mcp__claude_ai_Facebook_Ads__ads_insights_industry_benchmark` |
| Performance trend | `mcp__claude_ai_Facebook_Ads__ads_insights_performance_trend` |
| Extended Meta API (135 tools) | `meta-ads-135` (mikusnuz, installed, pending token) |
| Competitor research | `proxy-intell/facebook-ads-library-mcp` (planned install) |

## Hard Rules (ads_manager binding)

1. Never launch without CMO sign-off on creative.
2. Never exceed per-campaign budget cap without CFO sign-off.
3. Never modify a campaign owned by another in-flight task — check
   `touches` against campaign / ad-set IDs first.
4. Always pause learning-phase ad sets that hit $50 spend with
   < 0.5% CTR.
5. Always preserve a JSON snapshot of campaign / ad set / ad shape in
   the worktree (commit per change) so reviewer can diff.
6. **Confirm creative passes Brand-Truth gate before pulling to launch.**
   If asset hex / theme / mark deviates from canonical Brand-Truth doc,
   reject and reopen — never launch off-brand.

## Open Items

- [ ] WarpClip Page created + linked to ad account `1374787969502492`
- [ ] WarpClip Pixel installed on `warpclip.com` (CTO task)
- [ ] Conversions API server-side wired (CTO task, depends on Pixel)
- [ ] Business Manager created (consolidates ad account + page +
      pixel + future LINE / IG / Threads channels)
- [ ] Brand-Truth.md written for LungNote (before first LungNote campaign)
