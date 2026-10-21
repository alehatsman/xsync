# 12 — Plan of attack

A handoff. The spec (`01`–`06`, `10`, `11`) is complete and internally
consistent; this document is what to *do* with it, in order, and what was
found when the spec was first read against a real machine.

Nothing below changes a decision in [10](10-decisions.md). Three items
amend one.

---

## Finding — the mac bridge's iCloud path is probably wrong

Read-only probe, 2026-10-21, on the mac in this fleet:

```
~/Library/Mobile Documents/      — not present / not listable
~/Library/CloudStorage/
  GoogleDrive-aleh.atsman@gmail.com
  GoogleDrive-aleh.atsman@gmail.com (20-10-2026 20:33)
  iCloudDrive-iCloudDrive (19-10-2026 00:03)
```

Three consequences, none of them resolved:

1. **[04](04-platforms.md) hardcodes the mac bridge's iCloud side as
   `~/Library/Mobile Documents/…/notes`.** On this box iCloud surfaces
   through **File Provider under `CloudStorage/`** instead. That matters
   beyond the path string: [04](04-platforms.md#detection-per-surface) says
   macOS dehydration is detectable because `.<name>.icloud` stubs appear
   under Mobile Documents. A File Provider mount gives no such signal — the
   same blind spot the document already concedes for Drive on macOS. **If
   iCloud on this mac is File Provider only, the mac bridge has no
   placeholder detection at all**, and the only surface where the
   differentiating idea works is Windows.
2. The `(19-10-2026 00:03)` / `(20-10-2026 20:33)` suffixes are macOS
   reconnect/duplicate-account naming. **Two GoogleDrive roots exist.** A
   config that pins one of these paths is pinning something the OS renames
   on reconnect. Whatever `check` does about mount validation has to survive
   that, and the sentinel ([03](03-safety.md#sentinels)) is what makes it
   survivable — a renamed root is a missing sentinel, which is the correct
   failure.
3. The probe could not read *inside* those directories. Everything above is
   from the directory names alone. **Confirm before acting on it.**

This becomes **E2-mac** below. It is the same class of question as
[E2](11-open.md#e2--does-icloud-for-windows-even-expose-the-obsidian-container)
and it is equally load-bearing.

---

## Spec repairs — do these before the gate scripts

Doc-only. They change what the gate measures, which is why they come first.

### R1 — `.xsync/` lives inside a cloud-synced tree

Side B of the bridge is `gdrive\notes`. `.xsync/` is excluded from *xsync's*
walker ([03](03-safety.md#quarantine)) — Google's daemon does not read that
exclude list. So `.xsync/` is uploaded to Drive and fanned out to every
machine that mounts it.

What that breaks:

- **Quarantine replicates fleet-wide.** Every machine downloads every
  deleted file. [01](01-scope.md) says xsync is not a backup; this makes it
  one, badly, on every disk.
- **`tmp/` replicates.** Partially-written copy targets become real files
  the vendor daemon ships somewhere.
- **The sentinel stops being a fact.** [03](03-safety.md#sentinels) justifies
  the sentinel as "is this volume mounted, in one stat". A replicated
  sentinel appears on machines where `init` never ran, and survives in a
  stale local cache of a mount that is gone. That is precisely the state the
  guard exists to catch.

**Proposed fix, not yet decided:** state, trash and tmp move out of the tree
to `~/.local/state/xsync/<pair>/<side>/` (`%LOCALAPPDATA%` on Windows). Only
the sentinel stays in-tree, because only the sentinel has to be observable
*through* the mount to mean anything.

**Open against that fix:** quarantine stops being a same-filesystem rename
([02](02-model.md#apply) relies on rename atomicity), so every quarantine
becomes copy-then-unlink across devices — which puts `remove_file` on the
common path instead of only in the retention sweep. That is a real cost
against [03](03-safety.md)'s governing rule and it needs a decision, not a
patch. A middle option: trash stays in-tree, state and tmp move out, and the
in-tree trash gets whatever per-vendor ignore mechanism exists.

Whichever way it lands, it is **D13** and it is written down before code.

### R2 — D11's dedup is best-effort, not idempotence

[D11](10-decisions.md#d11--two-bridges-not-one) specifies that a bridge
hashes the losing content and skips writing a conflict file if a sibling
`*.conflict-*` with the same hash already exists, and claims this "makes the
second bridge idempotent with respect to the first."

It does not. The sibling only exists locally once the vendor daemon has
carried it across. Both bridges tick inside that window, so the expected
outcome of one genuine conflict is still two names on both sides.

The mechanism is worth keeping — it collapses the duplicate on a later tick,
and it is cheap. **The claim drops to that.** Amend D11's mitigation block;
do not remove it.

### R3 — `min_tracked = 1` is a no-op

[03](03-safety.md#circuit-breakers) lists it as catching "state says
hundreds, scan says none". At a threshold of 1 it fires only on a literally
empty scan, which the sentinel already caught one step earlier.

Either make it relative (a fraction of the previous scan's count, which
catches Drive mid-initial-sync — the case [D3](10-decisions.md#d3--a-sentinel-file-not-a-file-count)
says the count guard is *kept* for), or delete the row. Do not ship it at 1.

### R4 — flag only: "lazy hash" versus `Synced.hash`

[02](02-model.md#entry) says "a scan stats; it does not read" and that the
planner hashes only on both-sides-changed. But `Synced` carries a
non-optional `hash`, so every tracked path has been hashed at least once,
and a first run hashes both trees end to end.

That is almost certainly the right behaviour — it is a few thousand small
Markdown files, once. **But the document currently implies the opposite.**
Say what actually happens: hashes are computed on copy and on first-run
tracking, and the laziness is about *re-hashing on every scan*, not about
hashing at all.

No decision needed. One paragraph in `02`.

---

## The gate — instrumentation, then a week of waiting

[M0](../ROADMAP.md) is a measurement and it can end this project. The
instrumentation is two scripts and no Rust. Both read-only, both idempotent,
both re-runnable in a week for the second count.

### `tools/gate-windows.ps1`

On the main_pc Windows host. Answers **E1 and E2 in one run**:

- Does `%USERPROFILE%\iCloudDrive\` expose the Obsidian app container at all
  ([E2](11-open.md#e2--does-icloud-for-windows-even-expose-the-obsidian-container))
  — check this first and exit early if it does not, because a `No` ends the
  Windows bridge as specified.
- Walk both trees, count entries carrying `FILE_ATTRIBUTE_OFFLINE` (0x1000),
  `RECALL_ON_DATA_ACCESS` (0x400000), `RECALL_ON_OPEN` (0x40000). Report the
  three counts separately, not OR'd — they mean different things.
- Total entry count per tree, so the second run has a denominator.

### `tools/gate-mac.sh`

Answers **E2-mac**, which the spec did not know it had:

- Resolve the real iCloud and Drive roots under `~/Library/CloudStorage/`
  and report every candidate, including the date-suffixed duplicates.
- Report whether `~/Library/Mobile Documents/` exists at all.
- Count `.<name>.icloud` stubs in whichever iCloud root is real.
- Probe for *any* POSIX-visible dehydration signal on a File Provider mount:
  `stat` block count versus size, and `xattr -l` on a sample. A negative
  result here is a finding, not a failure — it confirms
  [04](04-platforms.md)'s known-unknown and tells you the mac bridge runs
  with placeholder detection off.

### Verdict format

Both scripts print one machine-greppable line per tree, in the shape
[05](05-cli.md#log) already uses. A human should not have to interpret the
output, because the second run happens a week later and memory is not a
measurement.

### Then

Pin both trees *available offline*. Wait a week. Re-run both. Over the same
week, count real conflicts produced by one-writer-at-a-time
([E4](11-open.md#e4--what-is-the-real-conflict-rate)).

**Zero placeholders both times → delete this repo and write a provision
component that runs `unison` on a timer.** [01](01-scope.md#why-not-unison-and-the-gate-that-could-end-this-project)
means it.

---

## The dotfiles issues, which currently have no owner

[M0](../ROADMAP.md) lists these as blocking deployment and nothing else in
this plan touches them. They are cheap, they are independent of the gate,
and they can be filed while the week runs:

- **`alehatsman/dotfiles#81`** declared Drive the only source of truth and
  explicitly rejected iCloud. xsync makes iCloud a co-equal copy. That is a
  rewrite of `#81`, not an addendum, and it needs its own issue before any
  of this touches a machine. It also changes the answers to `#97` (iPhone)
  and `#72` (gdrive-mirror).
- **`components/notes` stops being a symlink component**
  ([D12](10-decisions.md#d12--every-machine-gets-a-real-local-tree)). Its
  own issue. The symlink guards stop being the mechanism and the component
  is rewritten rather than amended.

File both on `alehatsman/dotfiles`. Neither is closed by this repo.

---

## Order, condensed

1. **R1–R4** — spec repairs, doc-only. R1 needs a decision and becomes D13.
2. **`tools/gate-mac.sh`** — E2-mac. Five minutes, and it can invalidate the
   mac bridge the same way E2 can invalidate the Windows one.
3. **`tools/gate-windows.ps1`** — E2 then E1, one run.
4. **Pin both trees. Wait a week.** File the two dotfiles issues meanwhile.
5. **Re-run both scripts.** Read the verdict honestly.
6. **M1**, or `rm -rf`.

Nothing in steps 1–5 is Rust. That is the point: the cheapest path to the
answer that decides whether any Rust gets written at all.

## What the next agent should not do

- **Do not start M1 to "get ahead" while the gate runs.** If the gate says
  unison, the planner is wasted work, and the planner is the part of this
  spec most likely to feel productive while being the least decidable.
- **Do not re-argue the settled list** in
  [11](11-open.md#settled--do-not-re-open-without-evidence). Bring a
  measurement or leave it alone.
- **Do not confirm the mac finding from this document.** It is directory
  names and one blocked `ls`. Run the script.
