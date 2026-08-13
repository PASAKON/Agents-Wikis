# Hermes Agent vs MoonieX ORG — case-by-case

**Surveyed:** 2026-08-13 by CTO, on Contabo (`vmi3371421`).
**Subject:** `github.com/NousResearch/hermes-agent`, installed at
`/home/hermes/.hermes/hermes-agent` (inspection copy at `/tmp/hermes-inspect`).
**Compared against:** `/Users/gob/Projects/Agents` (ORG runtime) + the two wikis.

---

## 0. Install state on Contabo (as found)

| Fact | Value |
|---|---|
| Install path | `/home/hermes/.hermes/hermes-agent` (managed install, `HERMES_HOME=/home/hermes/.hermes`) |
| Install method | official `install.sh` one-liner; `.install_method` present, bootstrap complete |
| Git | **1 commit, 1 author** (`eac1e25`, Victor Kyriazakos, 2026-08-12). Squashed snapshot clone, no upstream history. Upgrade path is `hermes update`, not `git pull`. |
| Provider wired | OpenRouter → `anthropic/claude-sonnet-5`, `max_tokens: 16384` |
| Secrets present | only `OPENROUTER_API_KEY` (shared with the Sompong/claudeflow key). **Claude Max subscription is NOT wired in.** |
| Running? | **No hermes process.** The `hermes`-owned procs in `ps` are the n8n container (UID 1000 collision, cosmetic). |
| Usage so far | `state.db` = 2 sessions / 3 messages. Effectively untouched. |
| Disk | 2.3 GB under `~/.hermes` (of 36 GB free on `/`) |
| Config oddity | `kanban.review_dispatch: true` is set but no dispatcher/gateway is running, so the board is inert |
| Side note | n8n container bind-mounts `/local-files` rw as UID 1000 = host `hermes`. Shared write surface, worth knowing before running Hermes with real credentials. |

---

## 1. Scale

| | Hermes Agent | ORG |
|---|---|---|
| Python files | 4,064 | 1,144 |
| Python LOC | **1,592,308** | 297,657 |
| TS/JS files | 2,174 | ~0 (no frontend) |
| Markdown | 1,517 (docs are product) | 3,067 (mostly `output/` + `knowledge/`) |
| Test files | 2,908 (~17k tests) | 234 |
| Commits | 1 (squashed) | 402 (real history) |
| Biggest single files | `cli.py` 880 KB, `hermes_state.py` 503 KB, `run_agent.py` 384 KB | `lib/db.py` 677 lines, `tools/delegate.py` 700 lines |

**Read:** Hermes is a product with ~5x our Python mass in single files alone.
We are an orchestration layer, not an agent runtime. Different weight classes,
different jobs.

---

## 2. Case-by-case

### 2.1 What the thing fundamentally is

- **Hermes** — a *self-contained agent runtime*. It owns the conversation loop
  (`run_agent.py::AIAgent.run_conversation`), the tool schema, the provider
  adapters, the context compression, the state DB. Bring an API key, get an agent.
- **ORG** — an *orchestration layer on top of Claude Code*. We do not own a
  conversation loop; Anthropic's harness is our loop. We own roles, tasks,
  worktrees, locks, wiki, and the human gates.

**Consequence:** they can swap models freely and we cannot; we get harness
upgrades free and they must build them.

### 2.2 Model / provider layer

| | Hermes | ORG |
|---|---|---|
| Providers | **34** plugin dirs (`anthropic, openrouter, bedrock, vertex, gemini, xai, deepseek, zai, kimi-coding, ollama-cloud, nous, copilot, azure-foundry, …`) | Claude (Max sub) + GLM-5.2 via Z.ai, routed by `lib/quota_router.py` |
| Switch cost | `hermes model` / `/model` — zero code change | env + `policies/agents.yaml` + launcher `.sh` (3 sources, see memory) |
| Subscription auth | OAuth-per-provider (`copilot`, `qwen-oauth`, Nous Portal) | Claude Max sub via the CLI's own login |

**Hermes wins outright.** This is their single strongest structural advantage.
Ours is a 3-file dance documented as a known trap.

### 2.3 Delegation / subagents

**Hermes** — `tools/delegate_tool.py` (4,356 lines):
- in-process child `AIAgent`, fresh conversation, own `task_id`, inherited toolsets
- `role="leaf"` (default) vs `role="orchestrator"`; depth capped `max_spawn_depth: 2`
- children **hard-blocked** from `delegate_task`, `clarify`, `memory`, `send_message`, `cronjob`
- batch/parallel mode, `max_concurrent_children: 3`
- `background=true` → returns a handle, result re-enters via a shared completion
  queue as a NEW turn (explicitly to avoid breaking role alternation + prompt cache)
- **explicit durability warning in their own docs**: background delegation is
  process-local; use `cronjob` or `terminal(background=True)` if it must survive restart

**ORG** — `tools/delegate.py` (700 lines):
- spawns a **real Claude Code TUI in an iTerm tab / tmux session** per DEV
- each DEV gets its own **git worktree + branch** (real filesystem isolation)
- blocks on a DB poll until the DEV calls `submit_report` (status → `review`)
- path-level `touches` locks + `depends_on`, collision pre-check before create
- mandatory visible kickoff ping (IRON §29)

| Dimension | Winner | Why |
|---|---|---|
| Cheapness / speed | Hermes | in-process, no worktree, no node_modules tax |
| Parallelism ergonomics | Hermes | one `tasks: [...]` call |
| Recursion safety | Hermes | typed roles + depth cap, enforced in code |
| Code isolation | **ORG** | worktree+branch means a bad DEV cannot corrupt main |
| Auditability / human watch | **ORG** | you can literally watch the DEV work |
| Durability across restart | tie | both are process-local and both admit it |
| Merge discipline | **ORG** | `merge_task` + cto-merge-checklist gate |

### 2.4 Task board / work queue

- **Hermes Kanban** — SQLite board (`~/.hermes/kanban.db`) + a **dispatcher loop**
  (default 60s inside the gateway) that reclaims stale claims, promotes ready
  tasks, atomically claims, and spawns the assigned profile. Verbs:
  `create/assign/link/comment/attach/complete/request-review/request-changes/block/archive/heartbeat/dispatch/daemon/gc`.
  Isolation: **board = hard boundary** (workers pinned via `HERMES_KANBAN_BOARD`),
  **tenant = soft namespace** inside a board. Auto-blocks a task after
  `failure_limit` (default 2) consecutive failures to stop spin loops.
  Web dashboard + systemd unit shipped.
- **ORG** — `state/tasks.db`: `tasks` (426 rows), `events` (2,685), `locks` (8),
  `c_level_sessions` (85). Rich task row (`touches`, `depends_on`, `worktree`,
  `branch`, `pid`, `tmux_session`, `ttyd_port`, `owner_role`, `model_hint`).
  **No dispatcher.** A C-level manually calls `delegate_task`.

| | Hermes | ORG |
|---|---|---|
| Autonomous pull of ready work | ✅ dispatcher daemon | ❌ human/CTO push only |
| Stale-claim reclaim | ✅ built in | ⚠️ `gc_stale_tasks.py` + `watchdog.py`, bolted on |
| Spin-loop protection | ✅ `failure_limit` auto-block | ⚠️ manual `max_iterations: 3` |
| Path-level conflict prevention | ❌ | ✅ `touches` + `locks` |
| Multi-tenant | ✅ board/tenant model | ❌ single tenant (MoonieX) |
| Full event log | ⚠️ runs/log | ✅ 2,685-row `events` table |

**This is the single biggest thing worth stealing.** Their dispatcher is exactly
the "queue drains itself" property we do not have.

### 2.5 Roles / org model

- **Hermes** — no role concept. Has *profiles* (multi-instance) + kanban
  *assignees*. Flat.
- **ORG** — 17 role definitions in `roles/` (`cto, cfo, cgo, cmo, developer,
  tester, web_designer, browser_operator, devops_engineer, security_engineer,
  data_analyst, prompt_engineer, ads_manager, content_strategist, video_editor,
  script_writer` + `_dev_shared`), levels `c/m/w`, per-role capability flags
  (`can_merge`, `can_push`, `can_write_wiki`, `can_delete_branch`), per-role
  model + effort + fallback.

**ORG wins outright.** Hermes has no answer to "who is allowed to merge."
Our permission model is a genuine differentiator.

### 2.6 Execution isolation

- **Hermes** — 7 terminal backends in `tools/environments/`:
  `local, docker, ssh, singularity, modal, daytona, vercel_sandbox`.
  Modal + Daytona give *serverless persistence*: the env hibernates when idle.
  Config already exposes `container_cpu/memory/disk/persistent`.
- **ORG** — git worktree + branch. Everything runs on the host, as the host user.

**Hermes wins on blast radius.** A rogue Hermes worker can be boxed in a
container on another machine; a rogue DEV of ours has the Mac. We compensate
with GateGuard hooks + human watching, which is weaker in kind.

### 2.7 Memory

| | Hermes | ORG |
|---|---|---|
| Store | `MEMORY.md` + `USER.md`, char-capped (2200 / 1375) | one fact per `.md` file + `MEMORY.md` index, ~90 entries |
| Recall | **FTS5 full-text search over every past message** (`hermes_state_search.py`, 114 KB) + trigram index + LLM summarization | `lib/recall.py` (212 lines) + `hook-recall.py` injecting at prompt-submit |
| Write trigger | `nudge_interval: 10` turns, `flush_min_turns: 6` — the agent nudges *itself* | model judgment, no nudge |
| Pluggable backends | **10**: honcho, mem0, supermemory, byterover, hindsight, holographic, openviking, retaindb | none |
| User modeling | Honcho dialectic user model | `user`-type memory files, hand-written |
| CJK/Thai search | native FTS5 CJK tokenizer (`native/fts5_cjk`) | n/a |

**Hermes wins on mechanism, ORG wins on curation quality.** Their memory is
searchable-everything; ours is hand-curated-and-typed with `**Why:**` /
`**How to apply:**` structure. Their self-nudge is the piece we lack.

### 2.8 Skills

| | Hermes | ORG |
|---|---|---|
| Built-in | **79** SKILL.md, 15 categories | 19 repo + 42 global = 61 |
| Optional (shipped, off by default) | **115** more | n/a |
| Hub / registry | agentskills.io open standard, `skills_hub.py`, `skills_sync.py` | `skills-lock.json` + wiki `skills-registry.md` |
| Agent creates its own | ✅ `creation_nudge_interval: 15` | ❌ human-authored only (`skill-author` skill) |
| Self-improve during use | ✅ `skill_manage(patch/edit)` | ❌ |
| Lifecycle | ✅ **Curator** (`agent/curator.py`, 2,019 lines): usage telemetry → active/stale/archived, auto-archive, pin, backup+rollback, never deletes | ❌ manual |
| Quality gates | `skill_linter.py`, `skills_ast_audit.py`, `skill_provenance.py`, `skills_guard.py` | description-writing convention only |

**Hermes wins decisively.** The Curator is a real closed loop: telemetry →
auto-transition → LLM review → archive, with hard invariants (only touches
`created_by: agent` skills, never deletes, pinned skills exempt).
Our 61 skills have zero usage telemetry and zero lifecycle.

### 2.9 Knowledge / doctrine

- **Hermes** — `AGENTS.md` **80,701 bytes** (single file, loaded as a context
  file). Contains a "Contribution Rubric", the **Footprint Ladder**, "Known
  Pitfalls", testing doctrine ("don't write change-detector tests"). Plus a
  Docusaurus site, 4 README languages (en/zh/es/ur), `docs/rfcs/`, `docs/ADR.md`.
- **ORG** — 2 wiki roots, 171 pages. `IRON-RULES.md` = 1,122 lines / 42+ numbered
  sections, ADRs in `decisions/`, playbooks, per-project pages, per-role pages.

**Roughly even, different shapes.** Their doctrine is one big file optimized for
an LLM's context; ours is a navigable wiki optimized for cross-session recall.
Their **Footprint Ladder** is a better-articulated decision rule than anything we
have:

> 1 extend existing code → 2 CLI command + skill → 3 service-gated tool
> (`check_fn`) → 4 plugin → 5 MCP server → 6 new core tool.
> *Pick the highest (least-footprint) rung that correctly solves it.*

Also worth quoting verbatim, because it is the thesis we keep re-deriving:

> "Per-conversation prompt caching is sacred… The core is a narrow waist;
> capability lives at the edges."

### 2.10 Scheduling

- **Hermes** — `cron/` module: duration (`30m`), phrase (`every monday 9am`),
  5-field cron, ISO one-shot. Per-job `skills`, `model`/`provider` override,
  `script` pre-run whose stdout is injected (`no_agent=True` = script *is* the
  job), `context_from` to chain job A's output into job B, `workdir` (loads that
  dir's `AGENTS.md`/`CLAUDE.md`), multi-platform delivery.
  Hardening: **3-minute hard interrupt**, catchup window (½ period, clamped
  120s–2h), 120s grace for missed one-shots, file lock `~/.hermes/cron/.tick.lock`,
  `skip_memory=True` on cron sessions.
- **ORG** — launchd plists (`com.mooniex.agents-watchdog`, `cfo-claude-usage`),
  the harness `/loop` + `CronCreate`, `runners/watchdog.py`, `auto_resume.py`.

**Hermes wins.** `context_from` job chaining and `script` pre-run are things we
have hand-rolled repeatedly. Their 3-minute hard interrupt is a discipline we
do not enforce anywhere.

### 2.11 Human surfaces / where you talk to it

- **Hermes** — one gateway process serving **22+ platforms**: telegram, discord,
  slack, whatsapp, signal, matrix, mattermost, teams, email, sms, irc, line,
  feishu, dingtalk, wecom, weixin, qqbot, google_chat, homeassistant, ntfy,
  bluebubbles, a2a, webhook, api_server. Plus a full Ink/React TUI, an ACP
  adapter (VS Code / Zed / JetBrains), an Electron desktop app, and a web
  dashboard. Voice memo transcription, cross-platform conversation continuity.
- **ORG** — iTerm tabs + tmux + ttyd, MoonieX Console (mobile web), and the
  separate `mooniex-line-poster` project. `dashboard.py`.

**Hermes wins by an order of magnitude.** LINE is a first-class shipped adapter
there; for us it is its own repo with its own history of pain.

### 2.12 Context / cost management

- **Hermes** — `compression.*` config block (threshold 0.5, target_ratio 0.2,
  `protect_last_n: 20`, `protect_first_n: 3`, proactive prune knobs),
  `trajectory_compressor.py` (70 KB), `docs/micro-compaction.md`,
  `agent/context_engine.py`, `prompt_caching.cache_ttl: 5m`, `credits_tracker.py`,
  `billing_usage.py`, `account_usage.py`, `/usage` and `/insights` commands.
  `tool_loop_guardrails` with warn/hard-stop thresholds on repeated failures.
- **ORG** — TOON encoding (IRON §38 / ADR 0010), harness-side auto-compaction,
  `cost-guardian` plugin skill, `claude-usage-monitor` widget, `quota_router.py`.

**Hermes wins on in-loop control, ORG wins on cross-session accounting.**
Their `tool_loop_guardrails` (warn after 2 exact failures, hard-stop after 5) is
a cheap, obviously-good guard we should copy into DEV spawning.

### 2.13 Security / approvals

- **Hermes** — `tools/approval.py`, `write_approval.py`, `path_security.py`,
  `url_safety.py`, `threat_patterns.py`, `tirith_security.py`, `osv_check.py`
  (dependency CVE), `self_repo_guard.py` (stops the agent nuking its own venv),
  DM pairing (`~/.hermes/pairing/`), container isolation,
  `docs/security/network-egress-isolation.md`, `plugins/security-guidance/`,
  `SECURITY.md` (15 KB, 2 languages).
- **ORG** — GateGuard fact-forcing hooks, `policies/permissions.md`, role
  capability flags, gitleaks pre-commit + CI, API key registry (IRON §34),
  `hook-browser-guard.py`, human CEO gates on paid APIs.

**Roughly even, opposite philosophies.** Theirs is code-enforced and
machine-checkable. Ours is process-enforced and human-in-the-loop. Two of their
pieces have no ORG equivalent and clearly should: **`self_repo_guard`** (we have
lost work to agents editing their own runtime) and **`osv_check`**.

### 2.14 Observability

- **Hermes** — `plugins/observability/langfuse` + `nemo_relay`,
  `docs/observability/monitoring.md`, `relay-shared-metrics.md`, session
  request dumps, `hermes logs --follow --level --session`, achievements plugin.
- **ORG** — `events` table (2,685 rows, full actor/kind/payload/ts), per-session
  log files, `dashboard.py`, live tab titles (IRON §32), `session-worktree`.

**Even.** Different targets: they instrument the *agent*, we instrument the *org*.

### 2.15 MCP

- **Hermes** — MCP **client** (`tools/mcp_tool.py`, OAuth manager, schema cache,
  stdio watchdog, dashboard OAuth) **and** MCP **server** (`mcp_serve.py`, 37 KB).
  Ships an optional-MCP catalog: comfy-cloud, figma, linear, n8n, unreal-engine.
- **ORG** — MCP servers we wrote (`org`, `lungnote`) consumed by Claude Code.

**Hermes wins.** Being an MCP *server* means Hermes can be a tool inside another
agent. Our org tools already are, so this is closer than it looks, but their
OAuth + watchdog + schema-cache layer is more mature.

### 2.16 Testing

- **Hermes** — 2,908 test files, ~17k tests, `scripts/run_tests.sh`,
  `run_tests_parallel.py`, live-test harness, conformance vectors, and explicit
  anti-patterns documented ("don't write change-detector tests", "tests must not
  write to `~/.hermes/`", "don't fake the host OS").
- **ORG** — 234 test files, mostly in DEV-owned project repos, not the runtime.

**Hermes wins decisively.** Our orchestration layer is thinly tested and we know
it (2026-08-06 audit).

### 2.17 Portability / deploy

- **Hermes** — one curl installer for Linux/macOS/WSL2/Termux, a PowerShell
  installer for native Windows (bundles MinGit), Docker + compose, Nix flake,
  Singularity, and serverless (Modal/Daytona). Runs on a $5 VPS.
- **ORG** — Mac-primary, tmux/iTerm-bound, Contabo mirror (Phase A only,
  ADR 2026-07-12; phases B–F pending). Wikis on Contabo are rsync snapshots,
  read-only, go stale.

**Hermes wins decisively.** Our runtime is welded to one laptop's window manager.

### 2.18 Licence / governance

- **Hermes** — MIT, Nous Research, active daily commits, Discord, public issues,
  4 translated READMEs, an automated PR triage sweeper with codified close reasons.
- **ORG** — private, single-operator (CEO) + CTO agents.

**Practical note:** MIT means we can lift code directly. The single squashed
commit means we cannot cherry-pick upstream fixes; it is take-a-snapshot or use
`hermes update`.

---

## 3. Verdict

### What Hermes does better than us (ranked by value to MoonieX)

1. **Kanban dispatcher** — the queue drains itself. We have no autonomous pull.
2. **Skill Curator** — usage telemetry + lifecycle + auto-archive. Our 61 skills
   are unmeasured.
3. **Provider portability** — 34 backends, one command. Our 3-source model
   config is a documented trap.
4. **Messaging gateway** — 22 platforms from one process, LINE included.
5. **Execution sandboxes** — docker/ssh/modal/daytona, not "run on the Mac".
6. **Cron with `context_from` + `script` + hard interrupt.**
7. **Session FTS5 search + self-nudged memory writes.**
8. **Tool-loop guardrails** (warn at 2 failures, stop at 5).
9. **`self_repo_guard` + `osv_check`.**
10. **Test mass** — 17k vs ~0 on the runtime.

### What we do better than Hermes

1. **Role + permission model.** 17 roles, `can_merge`/`can_push` flags. Hermes
   has no concept of "who may merge".
2. **Git worktree isolation per worker** + `touches` path locks + `depends_on`.
   Hermes workers share a filesystem.
3. **Merge review gate** (`merge_task`, cto-merge-checklist, three-dot diff review).
4. **The wiki as durable org doctrine** — IRON-RULES + ADRs + per-project pages,
   171 pages, versioned, C-level-write-only.
5. **Session discipline** — charter → one Entry Problem → exit gate. Hermes has
   sessions but no charter/DoD concept.
6. **Human-gate rigour** — GateGuard fact forcing, paid-API stop, CEO approval
   points. Hermes' approvals are about *danger*, ours are about *authority*.
7. **Business-domain knowledge** — brand/content/finance/SEO knowledge bases and
   MoonieX-specific skills. Hermes is domain-neutral.

### Honest framing

Hermes is a **better agent**. Our ORG is a **better company**.
They out-engineer us on the runtime by ~5x LOC and ~12x tests; we out-design
them on governance, isolation, and institutional memory. The two are not
competitors: Hermes is a candidate *runtime* underneath our org model, or a
parts bin.

### Recommended posture (CTO opinion, CEO decides)

**Do not migrate.** Rewriting the org onto Hermes throws away the role model,
the worktree isolation, the merge gate, and the wiki — the parts that are
actually ours — to gain a runtime we do not currently need, and it moves us off
the Claude Max subscription onto per-token OpenRouter billing.

**Do steal, in this order:**
1. A dispatcher loop for `state/tasks.db` (their kanban dispatcher, ~60s tick,
   stale-claim reclaim, `failure_limit` auto-block).
2. Skill usage telemetry + a curator pass over our 61 skills.
3. `tool_loop_guardrails` thresholds into DEV spawning.
4. A `self_repo_guard` equivalent for the Agents repo.

**Keep the install** as a research/parts box. It is 2.3 GB, idle, and costs
nothing while not running. If it is ever pointed at real work, wire it to its
own key (not the shared Sompong OpenRouter key) and box it in a container.

---

*Sources: `/tmp/hermes-inspect` (repo snapshot), `/home/hermes/.hermes/`
(live install), `/Users/gob/Projects/Agents`, `Agents-Wikis/IRON-RULES.md`.
All figures measured 2026-08-13, not quoted from marketing.*
