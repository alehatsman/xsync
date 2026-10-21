# 05 — Surface

## Commands

```
xsync check                 validate config, sentinels, mounts. exit != 0 on trouble
xsync init <pair>           write the sentinels on both sides. once, deliberate
xsync plan [<pair>]         print the actions a scan would take. reads nothing, writes nothing
xsync once [<pair>]         one scan + apply, then exit
xsync daemon                the loop, foreground. provision owns the service
xsync status                per pair: last scan, counts, drift, paused and why
xsync conflicts             list conflict files across all pairs
xsync resume <pair>         clear a tripped breaker (03)
xsync trash list|restore    the quarantine
```

`plan` and `once` are the same code path with the apply step switched off.
That is deliberate: a dry run that goes through a different function is a dry
run that is lying to you.

`daemon` runs in the foreground and logs to stdout. It does not fork, does
not write a pidfile, does not install itself. launchd, systemd and Task
Scheduler all prefer it that way, and provision owns all three
([01](01-scope.md)).

## Config

`~/.config/xsync/config.toml`, or `%APPDATA%\xsync\config.toml`. Managed by
provision; hand-editing it is the same mistake as hand-editing anything else
under a managed destination.

```toml
poll_interval  = "30s"
min_age        = "5s"
log            = "~/.local/state/xsync/xsync.log"

[[pair]]
name = "notes-bridge"
a    = 'C:\Users\aleha\iCloudDrive\Obsidian\notes'
b    = 'C:\Users\aleha\gdrive\notes'

a_placeholder = "skip"      # skip | hydrate | fail
b_placeholder = "skip"

max_delete_ratio  = 0.10
max_delete_count  = 50
max_conflict_count = 20
trash_retention   = "30d"

exclude = [
  ".obsidian/workspace*.json",
  ".obsidian/plugins/*/data.json",
  ".zk/notebook.db",
]

[[pair]]
name = "notes-wsl"
a    = "/mnt/c/Users/aleha/gdrive/notes"
b    = "~/notes"
# both plain trees — placeholder policy not permitted on a /mnt/c path (04)
```

The excludes in [03](03-safety.md#excludes-that-are-not-optional) are
compiled in. The `exclude` key **adds** to them and cannot remove them.
`.xsync/**` is not in either list because it is not a pattern — it is
skipped by the walker.

## Exit codes

| code | meaning |
|---|---|
| 0 | clean |
| 1 | config or usage error |
| 2 | a pair is paused (breaker tripped) |
| 3 | a side is unmounted (sentinel missing) |
| 4 | conflicts exist |

Distinct codes because the thing consuming them is a monitoring check, and
"something is wrong" is not an actionable alert. `xsync check` is the command
that is meant to be run on a schedule by something that pages you.

## Log

One line per event, timestamped, machine-greppable, no colour when not a tty.

```
2026-10-21T14:32:01Z pair=notes-bridge scan files=1284 placeholder=0 dur=84ms
2026-10-21T14:32:01Z pair=notes-bridge copy a->b inbox/thing.md 2.1kB
2026-10-21T14:32:01Z pair=notes-bridge CONFLICT projects/x.md kept=a other=x.conflict-mainpc-20261021T1432Z.md
2026-10-21T14:32:01Z pair=notes-bridge PAUSED breaker=max_delete_count plan_deletes=412 limit=50
```

A quiet run logs the one `scan` line. A daemon whose normal output is silence
is a daemon whose log you will actually read when it says something.
