# ROADMAP

## M0 — the gate that can end this project

**Before any code.** xsync's claim is that generic syncers mishandle
dehydrated files and that this fleet will hit that. Measure it:

1. On the Windows host, walk `iCloudDrive\…\notes` and `gdrive\notes` and
   count entries carrying `FILE_ATTRIBUTE_OFFLINE` /
   `RECALL_ON_DATA_ACCESS` / `RECALL_ON_OPEN`. A PowerShell one-liner;
   no Rust required.
2. Leave both pinned *available offline*. Re-count after a week.
3. Count, over the same week, how many real conflicts a one-writer-at-a-time
   workflow actually produces.

**If the placeholder count is zero both times**, the differentiating idea
never fires, xsync is a worse `unison`, and the correct outcome is to delete
this repo and write a provision component that runs unison on a timer. That
is a real possible ending, and it is cheap to reach from here —
which is why it is M0 and not M7.

**If it is non-zero**, build. Everything below assumes it is.

Also at M0, and **not** code:

- **Settled** ([10](docs/10-decisions.md)): two bridges (mac + main_pc's
  Windows host), every other machine gets a local pair, `~/notes` becomes a
  real directory everywhere.
- **Open:** `#81` declared Drive the only source of truth and explicitly
  rejected iCloud. xsync makes iCloud a co-equal copy — that is a rewrite of
  that decision, not an addition to it, and it needs an issue on
  `alehatsman/dotfiles` before any of this touches a machine. It also changes
  the answers to `#97` (iPhone) and `#72` (gdrive-mirror).
- **Open:** `components/notes` stops being a symlink component
  ([D12](docs/10-decisions.md#d12)). Its own issue.

## M1 — scan and plan

Walker, `Presence`, state snapshot, the pure planner, `xsync plan`. No
writes to anything but the snapshot. The fifteen-row table and its tests
land here. **This milestone is useful on its own**: `xsync plan` against the
real vault is the second measurement, and it is free to run.

## M2 — apply

Atomic copy, mtime propagation, excludes, `xsync once`. Copies only — no
deletes. A pair that only ever adds is a pair that cannot lose anything, and
running it in that state for a week is worth more than the week costs.

## M3 — refusing

Sentinels, `xsync init`, circuit breakers, quarantine, the retention sweep,
`xsync resume`. This is the milestone the program is for
([03](docs/03-safety.md)); deletes are not enabled before it lands.

## M4 — conflicts

Detection, both-sides conflict files, `xsync conflicts`.

## M5 — the loop

`xsync daemon`, `xsync status`, stability windows, exit codes, the log.

## M6 — placeholders

Windows attributes first, because that is the surface the gate measured.
macOS `.icloud` stubs second. Timeout-based detection for rclone/File
Provider last, or never.

## M7 — normalization and case

NFC comparison, collision refusal, the case rules
([04](docs/04-platforms.md)). Deliberately after the loop: it is a
correctness fix for a class that will show up in the real log, and the real
log is a better specification for it than this document is.

## M8 — provision

Components for the Windows Scheduled Task and the WSL systemd user unit, the
config template, and whatever `components/notes` has to become.

## Not scheduled

- rename detection by content hash
- a filesystem watcher as a poll accelerator
- any pair that is not notes
