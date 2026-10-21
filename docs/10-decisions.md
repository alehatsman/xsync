# 10 — The record

Decisions taken at spec time, before any code, with the reasoning that
produced them. **Nothing here has been tested against a real machine yet** —
which is itself the most important fact on the page, and the reason M0 in
[ROADMAP.md](../ROADMAP.md) is a measurement rather than a build step.

The other documents state current intent. This one is why they say it.

---

### D1 — No cloud API

**Alternative:** talk to Drive and iCloud directly, drop the dependency on
the desktop clients.

**Rejected because** it changes what the program *can* do wrong. A local file
mover can damage a directory. A credentialed API client can damage an
account, and it needs OAuth, token refresh, rate limiting, retry semantics
and a secret store before it moves its first byte. The desktop clients are
already installed, already authenticated, already running, and already
declared ([dotfiles#39](https://github.com/alehatsman/dotfiles/issues/39)).

**The rule it produced:** the fence is the network. xsync makes no outbound
connection, and any feature that needs one is a different program.

---

### D2 — Poll, never watch

**Alternative:** inotify / FSEvents / `ReadDirectoryChangesW`.

**Rejected as the primary signal** because three of the four surfaces cannot
provide it honestly. `/mnt/c` has no inotify at all. A FUSE mount emits what
the FUSE layer chooses. A macOS File Provider extension emits nothing a POSIX
watcher sees. A design where the correctness of sync depends on an event that
three platforms may not send is a design that fails silently, which is the
worst failure mode available.

**Polling fails loudly or not at all**: the scan either ran or it did not,
and `xsync status` knows which. A watcher may be added later to shorten the
next poll, never to decide that one happened.

---

### D3 — A sentinel file, not a file count

**Alternative:** `components/gdrive-mirror`'s guard — refuse if the source
has fewer than N files.

**Kept, but demoted to secondary.** The count guard answers "does this look
like an accident", which is a heuristic with a threshold that is wrong for
some tree. A sentinel answers "is this volume mounted", which is the actual
question, in one stat, with no tuning.

**What the count still catches** that the sentinel does not: a mount that is
present but half-populated, which is Drive mid-initial-sync. Both stay.

---

### D4 — Deletes resolve to conflict, not to delete

**Specifically:** `Gone` on one side against `Changed` on the other produces
a conflict and copies the surviving file back, rather than propagating the
deletion.

**Chosen because the two errors are not symmetrical.** Re-deleting a file
that came back is five seconds of mild annoyance. Recovering a file that was
deleted because the other side's edit lost a race is, at best, a trip to the
quarantine, and at worst nothing at all once the retention window closes.

**This will be annoying.** That is the intended cost, and the measurement
that would revisit it is a month of logs showing how often the row fires.

---

### D5 — A state snapshot file, not sqlite

**Rejected sqlite** because the only write is "replace the whole snapshot,
once per scan, after the actions already landed". Transactions, indices and
incremental updates all solve problems this access pattern does not have, and
a sqlite file inside a cloud-synced tree is its own hazard, which is why
[03](03-safety.md#excludes-that-are-not-optional) refuses to sync one.

**Revisit at ~10^5 entries per pair**, which a notes vault will not reach.

---

### D6 — The snapshot is written after apply

**Because the failure modes are not symmetrical.** A snapshot written first
and then interrupted records agreements that never happened — and the next
scan reads that as "both sides already match", which silently drops a real
change. A snapshot written last and then interrupted re-plans actions that
already ran, and every action is idempotent.

**The rule:** when a crash must corrupt something, make it corrupt the thing
that is recomputed rather than the thing that is trusted.

---

### D7 — `plan` and `once` are one code path

A dry run implemented separately from the real run is a dry run that can
disagree with it, and the disagreement surfaces exactly when it matters.
`plan` is `once` with the executor stubbed.

---

### D8 — No "newest wins"

**Rejected** because mtime is not a clock this program controls. Drive and
iCloud both set mtimes on their own schedule, DrvFs rounds, NTFS and ext4
disagree on granularity, and the two devices that wrote the conflicting
copies never agreed on the time anyway. "Newest wins" over those inputs is a
coin flip with a policy's name on it.

**mtime is still used** to decide which copy keeps the plain filename — a
presentation choice, where being wrong costs nothing, because both bytes
survive either way.

---

### D9 — `/mnt/c` may not be declared a cloud side

DrvFs does not surface Windows file attributes, and hydration requested
through it is the known-bad path. A `/mnt/c` path is a plain tree by
definition; the Windows-host daemon owns hydration for the Windows disk.
**Enforced in config validation**, not documented as advice — the failure it
prevents is a stub copied over a good file, which is silent.

---

### D10 — Daemons racing is allowed

Pairs share paths. The Windows-host bridge and the WSL pair both touch
`gdrive\notes`. The mac bridge and the Windows bridge both touch the same two
cloud namespaces, one hop removed. Nothing takes a lock.

**Allowed because every action is atomic and every scan re-derives truth from
the trees**, so a write landing between another daemon's scan and apply is
picked up on the next tick. A cross-boundary lock between a Windows process
and a WSL process — let alone between two machines — would be the most
complex thing in the program, in service of a race whose worst outcome is one
extra poll interval.

**What it costs, declared:** a change made in WSL reaches the iPhone after
two poll intervals plus two vendor sync delays.

---

### D11 — Two bridges, not one

**mac and the main_pc Windows host both run the iCloud↔Drive pair.** The
earlier spec said at most one host in the fleet should, and that was
overruled deliberately.

**Why two.** A single bridge is a single point of failure for the only link
the iPhone has. If the one host that owns it is asleep, powered off, or
mid-rebuild, the phone silently stops receiving notes — and silently is the
operative word, because nothing on the phone can tell "no new notes" from
"the bridge is down". Two bridges on two machines with different uptime
patterns means the phone keeps working when either is awake.

**Why it is safe.** Both bridges compute the same actions from the same two
namespaces. Where they overlap they write identical content to identical
paths, which converges. Where they disagree it is because one has seen a
change the other has not yet, and the next tick resolves it.

**What it actually costs, and the mitigation.** Duplicate conflict files: one
genuine conflict can produce `x.conflict-mac-….md` *and*
`x.conflict-mainpc-….md`, two names for one disagreement.

> **Mitigation, specified:** before writing a conflict file, xsync hashes the
> losing content and scans for an existing `*.conflict-*` sibling of the same
> base name with the same hash. If one exists, it writes nothing. Cheap, and
> it makes the second bridge idempotent with respect to the first.

**The measurement that would revert this to one bridge:** duplicate conflict
pairs appearing in the log despite the dedup, or any observed case of the two
bridges flapping a file between them. Neither is predicted; both are
detectable from `xsync status` alone.

---

### D12 — Every machine gets a real local tree

`components/notes` makes `~/notes` a symlink into the cloud mount, so every
read crosses DrvFs or FUSE. xsync replaces the link with a real directory and
a pair.

**Chosen because the cloud mount is the slowest filesystem on each of these
boxes, and it was serving every tool.** nvim, ripgrep and git now work
against ext4 or APFS; the mount is touched only by the one process written on
the assumption that it is slow and occasionally absent.

**What it costs:** a second copy of the vault per machine — a few megabytes
of Markdown — and the symlink guards in `components/notes` stop being the
mechanism that keeps `~/notes` honest. xsync's sentinels take that job
([03](03-safety.md#sentinels)), and the component has to be rewritten rather
than amended.
