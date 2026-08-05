# ADR 0014 — CXO sessions load a per-role MCP set under --strict-mcp-config

- **Date:** 2026-08-06
- **Author:** CTO (session #4911053b), at CEO request
- **Status:** Accepted, shipped `cfa85bc` (mooniex-agents main)
- **Supersedes:** nothing. Related: [0009 model routing](0009-model-routing-policy.md),
  [2026-07-12 org runtime to Contabo](2026-07-12-org-runtime-to-contabo.md)

## Context

The CEO reported the Mac was slow. Measurement, not guesswork, on the dev
box (MacBook Pro M1 2020, **8 GB RAM**, macOS 14.3.1):

| metric | measured |
|---|---|
| RAM unused | 143 MB |
| compressor | 2,142 MB |
| swapouts | 13,688 |
| 2 live claude sessions + MCP subtrees | **2,830 MB of 8,192** |

RSS is useless here — it fell from 1,873 MB to 568 MB in ten minutes as
macOS compressed idle pages. All figures above use `footprint -p <pid>`
(`phys_footprint`), which counts compressed pages.

Cost centre was MCP sprawl in C-level sessions: **13 servers, 1,489 MB**
for one CTO session whose `--allowed-tools` list only covers `mcp__org__*`
and `mcp__lungnote__*`. Each npx-based server costs twice — an `npm exec`
wrapper (~46 MB) holding the real node server (~43 MB).

Two compounding causes:

1. **No `--strict-mcp-config` on either CXO launcher.** `runners/dev_init.py`
   has passed it since it was written, so DEV spawns were never the problem
   — only the C-level ones.
2. **`enabledMcpjsonServers` declared in two files** — `~/.claude.json` and
   `.claude/settings.local.json`. Overlapping names spawn twice. Predicted
   6/6 against the live process list: four supabase refs (in both files) ran
   x2; `mooniex-coord` and `meta-ads-135` (one file) ran x1. Claude Code
   rewrites `settings.local.json` on its own, so this cannot be fixed by
   editing that file alone.

## Evidence

Usage audit over **199 transcripts / 6,478 logged MCP calls**
(2026-04-23..2026-08-06); 194 of 199 fall in the last 30 days, so all-time
equals current behaviour:

| MCP | calls | note |
|---|---:|---|
| claude-in-chrome | 4,918 | built-in, no process |
| org | 852 | core |
| lungnote | 285 | core |
| Drive / Gmail / FB Ads | 153 / 74 / 50 | remote connectors, no process |
| meigen | 43 | |
| ecc github | 39 | duplicate of `gh` CLI |
| supabase | 28 | |
| meta-ads-135 | 14 | |
| ecc exa | 5 | HTTP transport, no process |
| **ecc playwright** | **4** | duplicate of claude-in-chrome (4,918) |
| mooniex-coord | 2 | |
| **ecc context7 / memory / sequential-thinking** | **0** | |
| **polymarket** | **0** | arb project concluded |

## Decision

1. **One generator, `scripts/lib/cxo_mcp_config.py`**, shared by
   `cto-claude.sh` and `cxo-claude.sh`. Retires the drift between an inline
   heredoc and the committed `config/cto.mcp.json`, whose absolute Mac paths
   break on Contabo (`ROOT=/opt/mooniex-agents`).
2. **Per-role sets** — base `org, lungnote, mooniex-coord`; `+supabase` for
   cto/cfo/cgo; `+meta-ads-135` for cmo/cgo; `+meigen` for cmo.
3. **`--strict-mcp-config` on both launchers**, default on.
4. **Skip servers that are not installed** rather than spawning one that
   fails its handshake — this is what makes the launcher portable.
5. **Prune plugin MCPs with zero or duplicated usage** — dropped context7,
   memory, sequential-thinking, playwright from the active ECC install at
   `~/.claude/plugins/cache/ecc/ecc/2.0.0-rc.1/.mcp.json`. Kept `github`
   (used) and `exa` (HTTP, free).
6. **`spawn-cto.sh` warns** when another CTO chat is live and points at
   `--last`. Advisory only — it never kills. A quiet chat is usually waiting
   on a DEV, and reaping on idleness alone has silently killed live work.

## Consequences

- CTO: **13 servers → 4**, ~1,489 MB → ~290 MB per session.
- ECC prune: ~356 MB off **every** session, including plain CEO ones.
- CXO loses the `github` MCP under strict mode. `gh` is already the
  documented org path and `Bash` is in ALLOWED, so this consolidates on one
  GitHub route instead of two.
- Escape hatches, no code edit: `CXO_STRICT_MCP=0` restores the old
  inherit-everything behaviour; `CXO_EXTRA_MCP` / `CXO_SKIP_MCP` adjust a
  role's set; `CXO_NO_SESSION_WARN=1` silences the spawn warning.
- A plugin version bump installs to a new `cache/.../<version>/` directory,
  so the ECC prune does not carry forward — re-apply after upgrading ECC.

## Verification

`test_spawn_tab_routing` 28 PASS · `test_model_routing_consistency` ALL PASS
· `bash -n` clean on all three launchers · generator emits valid JSON for
all four roles with the Supabase PAT left as a `${VAR}` placeholder.

Baseline for the next measurement is in the commit body: 2,830 MB across two
sessions, 143 MB unused, compressor 2,142 MB.

## Not addressed

8 GB soldered RAM is the real ceiling; this buys headroom, it does not lift
it. The structural fix stays [org runtime to
Contabo](2026-07-12-org-runtime-to-contabo.md) — Phase A done, B–F pending.
