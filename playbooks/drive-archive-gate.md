<!-- Filed 2026-09-05 from ~/.claude/tools/drive-archive-gate.md per IRON-RULES §48 (CEO order). The copy next to the tool mirrors this page; edit here first. -->

# Drive archive gate (CEO 2026-09-05)

Rule for any agent asked to move data from the Mac to Google Drive. Nothing in
this file runs on its own: the daily disk watch only *reports* candidates, and
an agent moves them only after the CEO says go on a specific alert.

## What may move (and where)

| Source on the Mac | Destination in Google Drive (`ไดรฟ์ของฉัน/`) | Note |
|---|---|---|
| `~/Backups/**` | `Archive/Backups/<same relative path>` | VPS backups; keep folder names and dates |
| `~/Projects/Agents/output/**` | `Archive/Agents-output/<same relative path>` | generated deliverables (b-roll, prompts) |
| Claude transcripts | `Claude-Transcripts/<project>/<uuid>.tar.gz` | only via `prune_transcripts.py --archive`, never by hand |
| `winbox: Documents/CookieRunScript/play_rec/<take>/` (CEO's recorded Cookie Run takes) | `BACKUP/CookieRun Backup/play_rec/<take>.tar` + `<take>.manifest.json` | CEO-approved 2026-09-06; one tar per take (never loose frames), sha256 manifest, uploaded from the box with rclone (`gdrive:` remote, scope drive.file), verified with `rclone check`; steward brief `cookierun-bot/docs/DATA-STEWARD.md` |
| `winbox: Documents/CookieRunScript/play_rec/bot_session-*/`, `playset/*` (bot recordings, training sets) | `BACKUP/CookieRun Backup/bot_sessions/`, `.../playsets/` | same rules; proposed 2026-09-06, file only after the CEO OKs each batch |

Anything not in this table needs a new row approved by the CEO first. The
Drive folder is `/Users/gob/Library/CloudStorage/GoogleDrive-pass.gob1@gmail.com/ไดรฟ์ของฉัน`
(stream mode, 2 TB plan, ~1.8 TB free on 2026-09-05).

## Never touch

- `~/.claude/projects/*/memory/` (auto-memory), any live session's files
  (`~/.claude/sessions/*.json` lists them)
- `~/Library/Application Support/CloudDocs/session/i` — the iCloud store for
  the CEO's Desktop, not a cache
- `~/Pictures/Photos Library.photoslibrary` — only the Photos app or Drive's
  own Photos backup may read it
- `~/Desktop`, `~/Downloads`, `~/Movies` — CEO is consolidating phone data
  there (2026-09); parked until he says otherwise
- any `.git` directory, any `.venv` / `node_modules`
- Dropbox, iCloud Drive, or any other cloud folder as a destination

## How to move (copy, verify, then delete — in that order)

1. Confirm the go covers exactly this source path and this alert date.
2. `rsync -a --info=progress2 "<source>/" "<drive-dest>/"` — copy, never `mv`.
3. Wait until Drive has uploaded: the Drive menu-bar icon shows no pending
   items and every file in the destination opens. Do not proceed while
   "uploading" is still showing.
4. Verify: `cd <source> && find . -type f -exec shasum -a 256 {} + | sort -k2 > /tmp/src.sha`
   and the same in the destination; `diff` must be empty. Size alone is not
   verification.
5. Only now delete the source, and Empty Trash so the space actually returns.
6. Append one line per source to `~/.claude/logs/drive-archive.log`:
   date, source, destination, file count, bytes, sha-list checksum.
7. Tell the CEO over SomPong what moved, how much came back, and the
   destination path (`lib.telegram_out.send_to_ceo` from the Agents repo).

## Restore

Copy the folder back from the same Drive path to the original Mac path.
Transcripts: `python3 ~/.claude/tools/prune_transcripts.py --restore <uuid>`.

## Alerting

`~/.claude/tools/prune_transcripts.py --notify` runs daily 09:00 via launchd
(`com.gob.claude-prune-transcripts`). It messages the CEO (SomPong) when free
space < 20 GB, transcripts ready to archive ≥ 2 GB, gate candidates ≥ 2 GB,
or every Monday. Thresholds live in `~/.claude/prune-transcripts.json`.
