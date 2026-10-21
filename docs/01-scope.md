# 01 — Scope

xsync is **a bidirectional reconciler for two local directory trees, at least
one of which is a cloud client's mount.** The closed list:

1. **pair** — two local paths, declared, reconciled toward each other
2. **scan** — stat both trees on a timer; no event stream is trusted
3. **plan** — compare both scans against the last synced state, emit actions
4. **apply** — copy, delete, or flag, atomically, one action at a time
5. **absence** — a file that is dehydrated, unmounted or missing is a
   distinct fact from a file that was deleted, and is never confused with one
6. **refuse** — any scan that looks like an accident stops the pair cold

Six bullets. Everything not on that list belongs somewhere else.

## What it is

A **local** program. It opens files, stats directories, and writes files.
That is the entire set of things it does to the world.

- **pairs** — N of them, independent, configured ([05](05-cli.md)). A chain
  is two pairs sharing a path, written out, not inferred
- **state** — one snapshot per pair of what both sides looked like at the
  last successful reconcile. Without it, "created on A" and "deleted on B"
  are the same observation ([02](02-model.md))
- **placeholders** — a per-side policy for files the cloud client has not
  materialized: skip, hydrate, or fail ([04](04-platforms.md))
- **safety** — sentinels, circuit breakers, quarantine, and a conflict file
  that is never silently resolved ([03](03-safety.md))

## What it is not

**Not a cloud client.** No OAuth, no API, no network, no credentials, no
`reqwest`. If a side is not mounted, xsync refuses the pair; it does not go
and fetch it. This is the fence that keeps the program under a few thousand
lines, and every feature that would cross it is the wrong feature.

**Not a backup.** The quarantine ([03](03-safety.md)) is a window during
which a wrong delete is recoverable, not history. If you want to know what a
note said in March, that is git, or Time Machine, or Drive's own version
history — three things that already exist and are better at it.

**Not multi-machine.** xsync bridges two paths on *one* host. Getting those
paths to another host is Apple's job and Google's job, and they are paid for
it. A machine-to-machine xsync would need identity, transport, NAT traversal
and a trust model — that is Syncthing, and it is already written.

**Not N-way.** Pairs only. A three-corner topology is two pairs that share a
path, declared as two pairs. There is no graph, no solver, and no "primary".
Chaining costs one poll interval of latency per hop and the config says so
out loud, which is better than a topology engine that hides it.

**Not a conflict resolver.** When both sides changed, xsync keeps both and
tells you ([03](03-safety.md#conflicts)). It has no opinion about which of
your edits you meant. "Newest wins" across two filesystems whose clocks and
mtime granularity differ is a coin flip wearing a policy's clothes.

**Not real-time.** The poll interval is the floor on propagation. 30 seconds
by default. If you need sub-second, you need a different architecture and
probably a different problem.

**Not permission-preserving.** Content and mtime. Not mode bits, not owner,
not xattrs, not ACLs. One end of the primary pair is DrvFs over NTFS, where
those concepts do not survive the trip anyway; pretending otherwise would
produce a permanent diff that never converges.

**No rename detection, v1.** A rename is a delete and a create. It costs one
re-copy of a file that is, in this use case, a few kilobytes of Markdown, and
the quarantine means the delete half is never destructive. Content-hash
rename matching is cheap to add later and buys nothing until attachments are
large.

## How to reject a feature

Ask which of the six it is. If the answer needs a conjunction, it is not one
of them.

**"Talk to the Drive API so we don't need the desktop client."** No. That is
rclone, it is excellent, and the moment xsync holds a credential it becomes a
program that can delete your cloud account rather than a program that can
delete a directory.

**"Sync my whole Drive, not just notes."** Mechanically free — it is another
pair. Refused as a *default*: the primary pair's safety numbers
([03](03-safety.md)) are tuned for thousands of small text files that one
person edits one at a time. A 200 GB media tree with a dehydrated majority is
a different risk profile and should be declared as such, knowingly.

**"Watch the filesystem instead of polling."** Not as the only signal, ever.
`/mnt/c` has no inotify. A FUSE mount's events are the FUSE layer's opinion.
A File Provider extension emits nothing a POSIX watcher sees. Polling is the
correct answer on three of the four surfaces this runs on, and a watcher may
one day be added as an *accelerator* that shortens the next poll, never as
the thing that decides a scan happened.

**"Resolve conflicts automatically — newest wins."** No. See above. The
measurement that would change this is a count of real conflicts over a month;
if it is zero, the policy is untested and should stay maximally conservative,
and if it is not zero, you want to look at them.

**"A GUI."** No. `xsync status` and a log. The program's job is to be
uninteresting.

## The boundary, stated plainly

**If xsync dies, nothing happens.** Both trees stay exactly as they were.
Every action is atomic and independent; there is no half-applied state to
recover, only a stale snapshot that the next scan corrects.

**If a side is not mounted, the pair stops and says so.** An unmounted Drive
is an empty directory, and an empty directory is indistinguishable from "you
deleted everything" by content alone. xsync does not distinguish it by
content — it distinguishes it by a sentinel file ([03](03-safety.md#sentinels)),
which is a fact rather than a heuristic.

**xsync never deletes.** It moves to quarantine. The word *delete* appears in
the plan and in the log; the syscall never does.

## Why not unison, and the gate that could end this project

`unison` is mature, bidirectional, local-local capable, and has done this job
for twenty years. `rclone bisync` is in the box already. Either one would be
boring tech, which is the default this project has to beat.

The claim xsync makes is narrow: **those tools mishandle dehydrated files,
and on this fleet that will matter.** That claim is falsifiable, and it is
the M0 gate in [ROADMAP.md](../ROADMAP.md). If both sides of the real pair
are pinned *available offline* and a month of observation shows zero
dehydrated entries, then the differentiating idea never fires, the rest of
xsync is a worse unison, and **the correct outcome is to delete this repo and
write a provision component that runs unison.** That is a real possible
ending and the roadmap is written to reach it early rather than late.

## provision owns the machine state

The daemon is a long-running process on a machine, and this machine's state
is declared in `~/dotfiles` + provision. The Scheduled Task on the Windows
host and the systemd user unit in WSL are provision components, not something
an `xsync install` subcommand does behind provision's back. xsync ships no
installer; `components/notes` already knows how to speak about these paths.
