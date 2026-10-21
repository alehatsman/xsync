# 03 — Refusing

This is the document that matters. A sync program that is 99 % correct is a
program that eats a vault once a year, and once is the number that counts.

The governing rule: **xsync never issues `unlink`.** Not for a delete, not
for an overwrite, not for a conflict. Every destructive-looking action is a
rename into a quarantine with a retention window. Deletion in this program is
something a retention sweep does to a file that has been sitting in a dated
folder for thirty days, and nothing else.

## Sentinels

Each side of a pair carries `.xsync/pair-<name>`, a small file written once
at `xsync init`. Before every scan, both must exist and contain the pair's
id.

**Missing sentinel = the side is not mounted. Fail the pair. Do nothing.**

This is the primary mount guard, and it is a fact rather than a heuristic.
An unmounted Drive, a signed-out iCloud, an rclone mount that dropped, a
WSL guest that cannot see `/mnt/c` — all of them present as *an empty
directory that still exists*, which no amount of content inspection can
distinguish from a tree you emptied on purpose. A sentinel distinguishes it
in one stat.

`components/gdrive-mirror` learned the content-inspection version of this
lesson the expensive way and guards on a file *count*. That guard is kept
here as a secondary, below, but the sentinel is the one that is right.

## Circuit breakers

Applied to the plan, before any action runs. Tripping one **stops the whole
pair** — not the offending action, the pair. A plan that is wrong about 400
files is a plan that is wrong, and the twelve actions in it that looked fine
are not trustworthy either.

| breaker | default | catches |
|---|---|---|
| `max_delete_ratio` | 0.10 | a side that emptied itself |
| `max_delete_count` | 50 | the same, on a small tree where a ratio is noise |
| `max_conflict_count` | 20 | a clock skew or normalization bug conflicting everything |
| `min_tracked` | 1 | state says hundreds, scan says none |

A tripped pair is **paused and stays paused.** It does not retry on the next
tick, because the next tick will trip the same breaker and the one after that
will too, and a daemon that logs the same catastrophe every thirty seconds
is a daemon nobody reads. `xsync status` shows it; `xsync resume <pair>`
clears it, and that is a human deciding, which is the point.

## Quarantine

```
<side>/.xsync/trash/2026-10-21/notes/inbox/thing.md
```

Dated, path-preserving, on the side the file was removed from. `xsync trash
list` enumerates; `xsync trash restore <path>` puts one back.

Retention defaults to 30 days, swept at the start of a scan. The sweep is the
only code in xsync that calls `remove_file`, it only ever touches paths under
`.xsync/trash/<date>/` where `<date>` is older than the window, and it is the
one function that deserves a paranoid test.

`.xsync/` is excluded from scanning on both sides, always, unconditionally,
and not via the user's exclude list where it could be edited out.

## Conflicts

Both sides changed, content differs. xsync **keeps both and resolves
nothing.**

```
inbox/thing.md                              <- the newer mtime
inbox/thing.conflict-mainpc-20261021T1432Z.md   <- the other one
```

Written **to both sides**, so the conflict is visible from whichever machine
you next sit at — rather than only on the one that happened to lose.

Which copy keeps the plain name is decided by mtime, and that is a
presentation choice, not a correctness claim. Both bytes survive either way.
The filename carries the hostname because a conflict's first question is
always *which machine was that*.

Conflict files are ordinary files. They sync like anything else, they show up
in Obsidian, and they are deleted by you. xsync will never clean one up.

## Excludes that are not optional

Some paths are conflict generators and must never be synced. These ship as
defaults and the reason is specific in each case:

| pattern | why |
|---|---|
| `.obsidian/workspace*.json` | rewritten on every pane focus on every device; syncing it means a permanent conflict and a vault that reopens on the wrong note |
| `.obsidian/plugins/*/data.json` | per-device plugin state, same shape of problem |
| `*.gdoc` `*.gsheet` `*.gslides` | Drive shortcuts, not documents. Zero-byte JSON pointers |
| `.DS_Store`, `Icon\r`, `desktop.ini`, `Thumbs.db` | per-platform noise |
| `~$*`, `*.tmp`, `*.swp`, `.#*` | in-flight editor state |
| `.xsync/**` | xsync's own bookkeeping. Hardcoded, not overridable |

The first is the one that would have bitten within a day. It is the reason
this table exists as a shipped default rather than as advice.

Any rebuildable index a tool drops inside the vault belongs here too — a
sqlite file copied mid-write is a corrupt sqlite file, and a regenerable one
is not worth syncing even when it is intact. Add it when the tool arrives.

## What is deliberately not guarded

**Content.** xsync does not know what a note is, does not parse Markdown, and
will happily propagate a file you truncated to zero bytes on purpose. That is
an edit. Guarding against bad edits is version control's job, and one is
available.

**Disk space.** A copy that fails on ENOSPC fails that action, logs, and
leaves both sides as they were. No pre-flight capacity check; the atomic-
rename discipline makes the failure boring.
