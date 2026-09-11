# Plan — move the org's centre of gravity to Contabo

- **Date:** 2026-09-11
- **Status:** Proposed, awaiting CEO approval. Nothing here has been executed.
- **Owner:** CTO
- **Relates to:** ADR 0024 (split-brain task database), IRON §50

## What the CEO asked for

> "ต่อจากนี้ฉันอาจจะใช้ MAC น้อยลง เนื่องจากปัญหาด้าน Memory อาจจะกระจาย Worker CTO
> อื่นๆ ไปไว้ที่อื่นมากขึ้น คิดว่า CONTABO คือจุดที่ดีที่สุด เพราะเขาออนไลน์ตลอด และที่สำคัญคือ
> ต่อให้ไม่เปิด MAC หรือ Window ก็ยังใช้งานทำงานผ่านมือถือได้"

The goal is **work from a phone with the Mac and the Windows box switched off.** Everything
below is judged against that sentence and nothing else.

## Measured first, 2026-09-11

### Contabo can hold six working sessions, not more

| | measured |
|---|---|
| RAM | 7.8 GB total; 2.2 GB is baseline (8 docker containers, n8n, traefik, tailscaled, fail2ban, org runners) |
| usable for sessions | ~4.7 GB after reserving 1 GB for page cache |
| a session actively working | ~720 MB (claude + its MCP server + wrappers, RSS plus swapped-out anon) |
| a session parked | ~300 MB |
| **safe concurrent working sessions** | **6.** 7 is degraded, 8 exceeds RAM and swaps |
| realistic phone mix | 2 working + 10 parked = 12 |
| CPU | 4 vCPU @ 2.0 GHz; the one active session sustains 17 % of a core |
| disk | 72 GB, **29 GB free** |

Independent confirmation of the ceiling: the `user@0.service` cgroup that every tmux/claude
session lives in reports a measured `MemoryPeak` of 5.58 GiB.

**The honest headline: Contabo has 7.8 GB and the Mac has 8 GB.** Moving does not buy more
memory. It buys two different things — memory that is not shared with Chrome, Photos and
macOS, and a machine that never sleeps. Both are real; neither is "more RAM".

### Almost nothing has to move

Mac `/Users/gob/Projects` totals 18 GB, but the repos with commits this month are:

| repo | on Contabo today | note |
|---|---|---|
| Agents | ✅ `/opt/mooniex-agents` | the org runtime itself |
| Agents-Wikis (`org:`) | ⚠️ rsync snapshot, **no `.git`** | a `wiki_write` there is destroyed by the next sync |
| LLMs (`mooniex:`) | ⚠️ rsync snapshot, **no `.git`** | same |
| mooniex-claudeflow | ✅ `/root/projects/` | |
| cookierun-bot | ❌ 5.7 GB | needs winbox's GUI and the game; does not belong on a VPS |
| comfy-runpod-worker | ❌ 2.1 GB | drives a rented GPU; the work is remote anyway |

**So the gap is not "repos are missing".** Of six live repos, three are already there, two
genuinely belong on other hardware, and the remaining problem is that the two wikis are
snapshots rather than checkouts. That is a much smaller job than it first appears.

### Two live repos exist in exactly one place on earth

`cookierun-bot` (5.7 GB) and `comfy-runpod-worker` (2.1 GB) have **no git remote at all.**
They are not on GitHub and not on any other machine. This is a data-loss risk today, entirely
independent of this plan, and it is the most urgent item on this page.

### What the code actually blocks

`tools/delegate.py:729` is a hard stop:

```python
if host_cfg.get("os") != "windows":
    raise NotImplementedError(... "only winbox is wired in Phase 1")
```

and the string `"mac"` is used in **13 live-code sites across 6 modules** as a sentinel meaning
*"here, locally"* rather than *"the Mac machine"* — `tools/delegate.py:71, 881, 897, 974`;
`runners/watchdog.py:376, 477, 478`; `runners/branch_poller.py:144, 210, 276`;
`tools/gc_stale_tasks.py:165`; `tools/worker_reap.py:639`; `lib/config.py:168`.

The replacement already exists and is unused: `runners/worker_init.py:131` `current_host()`
reads `ORG_HOST`, but only `windows/spawn-worker.ps1:210` ever sets it.

Two pieces of good news: `tools/worker_reap.py:562-563` already has a Unix kill branch
(`tmux kill-session` + `kill`), and `runners/relay_mcp_server.py:1098, 1296` is already
two-host aware.

---

## The plan, in four phases

Each phase is independently useful and independently revertible. Stop after any of them.

### Phase 0 — back up what exists in one place (do this first, regardless)

Not part of the migration. It is simply the largest risk currently on the table.

1. Create GitHub repos for `cookierun-bot` and `comfy-runpod-worker` and push them.
   Their large data directories stay gitignored; the point is the code and history.
2. Push the Contabo checkout's 21 unpushed commits (already queued with the CEO, waiting
   on another agent to finish).
3. `Agents/.git` is 3.5 GB against a 2.8 GB working tree — worth one look for bloat, but not
   a blocker.

**Cost:** under an hour. **Risk:** none. **Reversible:** trivially.

### Phase 1 — make Contabo self-sufficient for the work that already lives there

This is the phase that delivers the CEO's sentence. It contains no hub/spoke rewrite.

1. **Turn the two wikis into real git checkouts** on Contabo, replacing the rsync snapshots.
   Today a C-level session there can write a wiki page and lose it silently on the next sync.
   This is a correctness bug, not just a migration step.
2. **Deploy the session-visibility fix** merged tonight (`main` at `f95df37`), then restart
   the two Contabo sessions so they register in the app. `--remote-control` is applied at
   launch and cannot be retrofitted to a running session.
3. **Retire the stale process** found during the audit: a `claude setup-token` owned by user
   `secretary`, running 13 days, holding 126 MB. CEO decision, not a steward action.
4. **The browser question is now answered, and the answer is no.** Leave
   `max_browser_operators: 0` for Contabo as it is. See the section below.

**Cost:** a few hours. **Risk:** low; step 2 touches a box with live sessions, so it waits for
the CEO's go. **Reversible:** yes.

### Phase 2 — one database, one hub

This is what permanently kills the class of bug in ADR 0024, where the same task id was found
running on two machines at once with no row on either hub that knew about both.

1. Replace the `"mac"` sentinel with `current_host()` at all 13 sites. Small in lines, hot in
   path — six modules share the idiom, so it needs one careful pass and its own tests.
2. Make exactly one machine the hub for each spoke. Contabo becomes the hub; the Mac stops
   spawning onto winbox and stops running a second watchdog against a second database.
3. Keep the existing relay for the reverse direction: `runners/mac_agent.py` (launchd
   `com.mooniex.mac-agent`) already dials out from the Mac and drains a queue on Contabo, and
   its own docstring records that Contabo cannot reach the Mac **by design**, with sshd closed
   there as a security property.

**Cost:** one worker, most of a day, plus review. **Risk:** medium — it touches the reap and
poll paths that were only just repaired tonight. **Reversible:** yes, it is one branch.

### Phase 3 — the Mac as a spoke (optional, and probably unnecessary)

Only worth doing if the CEO wants Contabo to drive work on the Mac while the Mac is on.

1. A new `scripts/spawn-worker-remote.sh` (~120 lines). It is a **new file, not an adaptation**
   of the existing Mac launcher, which assumes it is running locally.
2. Remove the `os != "windows"` hard stop and generalise `_remote_sha256` (`Get-FileHash`),
   the remote mkdir (`New-Item`), `_ps_quote`, and `remote_pid_alive` (`tasklist`), all of
   which are PowerShell-only today.
3. The Mac needs an inbound SSH channel. **Do not open macOS Remote Login to the internet.**
   Both machines are already on the same Tailscale network — the Mac is `macbook-pro--gob`
   at `100.64.2.37`, Contabo is `mooniex-contabo` — so this is a private WireGuard link, and
   Tailscale's own SSH should be preferred over enabling the system service. Enabling anything
   here needs the CEO's admin password and is theirs to run, not the CTO's.

**Cost:** a worker plus a careful review. **Risk:** medium-high; it is the only phase that
weakens a deliberate security property. **Reversible:** yes, but the SSH exposure is a
standing change, not a commit.

**The cheaper alternative to this whole phase:** when the CEO is sitting at the Mac, open the
session *on Contabo* from the Mac's terminal instead. No inbound SSH, no new launcher, no
sentinel rewrite. The only thing lost is a worker editing a Mac-only repo — which Phase 0 and
Phase 1 have already reduced to `cookierun-bot` and `comfy-runpod-worker`, both of which need
other hardware anyway.

---

## Disk is not the blocker, measured

29 GB free, and **~18 GB of that is genuinely reclaimable**: a 12.95 GB docker build cache
(131 entries, zero active, last used two months ago), ~3.4 GB of journal if vacuumed to
500 MB, and 1.8 GB in `/tmp/claude-0`. Reclaiming would take the box from 29 GB to ~47 GB.
Nothing has been reclaimed — Contabo is production and every write there needs the CEO's go.

Against that, the repos worth cloning cost about **3 GB** in total (`mooniex-claudesign` 2.5 GB,
of which 1.8 GB is `.git`, plus four small ones). Five repos named in `Agents/CLAUDE.md` as
"not on Contabo yet" are **not on the Mac either** and could not be sized; they would have to
come from GitHub if they exist there at all.

## What will never work on Contabo, whatever we do

- **Browser work — and this is the important one.** Contabo has no browser binary at all: no
  chrome, chromium or firefox, no Playwright or Puppeteer cache. `xvfb` and the headless
  shared libraries are present, so one *could* be installed for roughly 400 MB of disk and
  250-700 MB of RAM per lane.
  **But installing one would not restore `browser_operator`.** The org drives browsers through
  `mcp__claude-in-chrome__*`, which attaches to a **desktop Chrome carrying the CEO's
  logged-in profile** — Higgsfield, Google Flow, Drive. A headless Chromium on a VPS holds
  none of those sessions. Browser work therefore requires a machine where the CEO is signed
  in, which today means winbox or the Mac, and no amount of VPS configuration changes that.
- **Cookie Run.** It needs the game running in BlueStacks on winbox, with a real desktop.
- **Anything GPU.** Contabo has none; that work is RunPod's and stays remote.
- **macOS-only automation** — iTerm tab routing, AppleScript, computer-use on the Mac's screen.

### What this costs the CEO's goal, stated plainly

"Work from a phone with the Mac and the Windows box off" holds for **org, wiki, planning,
code and deploy work**. It does **not** hold for the film pipeline: every Higgsfield
generation, every harvest, every Drive filing step runs through a logged-in Chrome on winbox.
Those remain winbox work, and winbox must be on for them. This is a property of the vendors'
login model, not of our architecture.

None of these is a reason not to proceed. They are a reason to keep winbox and the Mac as
capable machines rather than to decommission them.

## Recommendation

Do **Phase 0 now** regardless of the decision, because two live repos exist in one place.

Then do **Phase 1**, which is small, and measure whether the CEO's actual daily work becomes
possible from a phone. It very likely does, because three of the six live repos are already
on Contabo.

Treat **Phase 2** as the real project, and **Phase 3** as optional — with the terminal-out
alternative tried first, since it costs nothing and gives up only the two repos that belong
on other hardware.
