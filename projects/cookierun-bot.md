# cookierun-bot — the Cookie Run Classic money farm that learns

**Repo:** `/Users/gob/Projects/cookierun-bot` (Mac) · box checkout `C:\Users\UsEr\cookierun-bot` (winbox, BlueStacks; not a git clone — deploy = scp).
**Owner:** CEO plays and answers; CTO runs the loop. Project knowledge lives in the repo:
`knowledge/cookierun-game-mechanics.md` (game facts, tagged [CEO]/[measured]/[fandom]),
`docs/APP-SUPERVISOR.md` (the app is the supervisor and the interface), `docs/DATA-STEWARD.md` (data lifecycle).

## Standing rules (CEO, dated)
- **2026-09-06 — Cookie Run Script is the only interface.** Every workflow the CTO runs is a button in the app
  and a function on its local pipe (`127.0.0.1:8794`); the CTO drives the app through `tools/appctl.py`, the
  window shows `[ท่อ:CTO] เรียก …`. Button first, pipe second, script never alone.
- **2026-09-06 — ESC above everything; no window = no bot.** ESC/F12 kills the bot and holds every launch
  until ▶; the bot runs inside a kill-on-close Job Object and stops itself without the app's heartbeat.
  The hotkey is machine-wide (pynput): an ESC pressed in any program stops the bot. The hold is cleared only
  by ▶ or an explicit CEO order; the CTO never clears it on its own.
- **2026-09-06 — Data steward.** Warn, never auto-delete (`tools/disksense.py`, daily). Back up to Drive
  (`BACKUP/CookieRun Backup/`, gate row in `playbooks/drive-archive-gate.md`) before deleting; one tar per
  take; takes kept locally as MP4. RunPod volume `cookierun-train` (EU-RO-1, 10 GB) holds only the current
  training tarball. Data stays on the Windows box.
- **2026-09-07 — When the CEO uses the PC, the CTO does not touch it.** No builds, archives, uploads or
  relaunches until the CEO says the box is free; heavy work only in CEO-use windows.
- **Goals (2026-09-06):** survival first (falls > hits), coins are the scoreboard, nights unattended, days tuned
  together; a richer stage only when EP1 hits ≈ 0.

## State on 2026-09-07
- Champion **v5**, candidate **v7**. Pooled rounds so far: 09-06 v5 n=73 coins 67.5k / v7 n=14 coins 60.0k;
  09-07 v5 n=3 49.7k / v7 n=3 58.2k. v7 falls at least as often as v5 wherever the fall counter existed
  (it was added mid-day 09-06, so v5's 09-06 falls are undercounted). No promotion; the 60-round A/B resumes
  when the box is free — needs ≥20 rounds per arm with the fall counter.
- **Incident 2026-09-07 04:31-07:21:** Windows 11's "Select a default app for .pdf files" picker (Adobe
  Acrobat's make-me-default nag after its overnight update) sat over the lobby for 2 h 50 min. The navigator
  refused the unknown dialog (correct for GAME dialogs, wrong here) — 17 navigations, 665 frames, zero runs —
  and the CTO's watchdog only checked the pipe. The CEO found it and closed the app.
  Fixes shipped: `core.foreign_window_over_game()` + `close_foreign_window()` (minimise; WM_CLOSE on the third
  return; never a console) at both refusal sites in `engine.navigate_to_run` (82e327d, deployed, live);
  app stall line "🔴 ค้าง N นาที ไม่มีรอบจบ" past 25 min without a finished round (00dddb5, needs an exe
  rebuild); the CTO watchdog now measures the age of `rounds.jsonl`. Source fix: the CEO picks Acrobat once.
- Overnight 09-06→07: 52 sessions / 0 runs from a `spec` NameError in play_model (fixed b7f33ca). Rule: every
  play_model change gets a dry smoke on the box before a launch; the app preflight (in progress) makes that
  automatic and hash-gated.
- Open: exe rebuild (stall line + clear_hold relay + guard fixes); drive.file re-auth; take-3 archive blocked on
  the CEO's own Google OAuth client_id (rclone's shared id hits rateLimitExceeded); 🧹 bot_sessions to Drive;
  ☁ RunPod training as a button; v8a sweep 2/4/8 frames (+press-age, lead 2) — ~$1.2 on the v7a tarball
  already on the volume, awaiting the CEO's spend OK.

## Scaling options reviewed 2026-09-07 (CEO asked how to run many instances without using his screen)
- **Cloud GPU VM (T4 ~$400/mo):** rejected — Android in a container/VM cannot use an NVIDIA GPU (Mesa is
  Intel/AMD only) and AWS/GCP/Azure VMs lack nested virtualisation for BlueStacks; egress $0.08-0.12/GB.
- **Bare metal with an Intel iGPU + Windows + BlueStacks (same stack):** OVH Singapore Rise (~€100-150/mo),
  Vultr bare metal Tokyo/Singapore (~$120-185/mo, hourly), Hetzner EX44 Germany (~€44/mo, 200 ms only matters
  for viewing). Needs a virtual display driver (Parsec VDD) and Parsec/VNC instead of RDP; datacenter IP risk.
- **Old Android phones + scrcpy/ADB, PC as the brain:** the cheapest physical farm — broken-screen phones
  ~฿500-900 each, 10-phone rack ~฿10k; +60-90 ms latency so v7 must be re-DAggered on the phone pipeline
  (`--lead-frames` compensates); one phone model per farm (aspect ratio, thermal profile, Xiaomi needs the
  "USB debugging (security settings)" toggle; Samsung does not). Screen off via `scrcpy --turn-screen-off`.
- **Emulator farm on one used PC (BlueStacks multi-instance, 8-10 per ฿15-20k box):** same stack, same
  scrcpy/ADB backend; ~฿1,500-2,500 per instance.
- **Cloud phones (Redfinger VIP Thailand ~$4-6/mo, VMOS Cloud $4.99/30 d with root+ADB):** exist under ฿500/mo,
  but only usable with the bot running ON the phone (own APK: Accessibility input + MediaProjection capture +
  a distilled small model) — streaming to the PC brain adds 100-300 ms and is a non-starter. Trial proposed:
  $0.99 Redfinger day + $4.99 VMOS month ≈ $6, awaiting OK.
- **Zero-hardware product:** the APK on the customer's own phone — cheapest, highest ban/support risk.
  Any customer-facing bot service must disclose the game-ToS/ban risk, isolate per-slot sessions, never hold
  customer passwords (they log in themselves via a remote screen), and treat the collected run data under PDPA.
