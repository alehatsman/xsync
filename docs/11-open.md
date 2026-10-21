# 11 — What is not known yet

This repo is a specification with no code. Everything in `01`–`06` is a
design that has never met a filesystem, and the honest status of most of it
is *plausible*. This document is the handoff: what to measure, in what order,
and what not to re-argue.

## Settled — do not re-open without evidence

These were decided deliberately. Each has its reasoning in
[10](10-decisions.md). Reopening one costs nothing if you bring a
measurement, and wastes a session if you bring a preference.

- no cloud API, ever ([D1](10-decisions.md#d1--no-cloud-api))
- polling, not watching ([D2](10-decisions.md#d2--poll-never-watch))
- delete-vs-edit resolves to conflict ([D4](10-decisions.md#d4--deletes-resolve-to-conflict-not-to-delete))
- state snapshot written *after* apply ([D6](10-decisions.md#d6--the-snapshot-is-written-after-apply))
- two bridges, mac and main_pc ([D11](10-decisions.md#d11--two-bridges-not-one))
- `~/notes` becomes a real directory on every machine ([D12](10-decisions.md#d12--every-machine-gets-a-real-local-tree))

## Experiments, in order

### E1 — does the placeholder claim hold? *(the gate)*

The one thing that justifies this program over `unison`. On the main_pc
Windows host:

```powershell
Get-ChildItem -Recurse -Force 'C:\Users\aleha\gdrive\notes',
  'C:\Users\aleha\iCloudDrive\Obsidian\notes' -File |
  Where-Object { $_.Attributes -band 0x401000 -or $_.Attributes -band 0x40000 } |
  Select-Object FullName, Attributes
```

`0x1000` OFFLINE, `0x400000` RECALL_ON_DATA_ACCESS, `0x40000`
RECALL_ON_OPEN. Count now, pin both trees *available offline*, count again
after a week.

**Zero both times → stop. Delete this repo, write a provision component that
runs `unison` on a timer, and close the loop honestly.** That outcome is a
success of the gate, not a failure of the project.

### E2 — does iCloud for Windows even expose the Obsidian container?

**Assumed, never verified, and load-bearing.** Obsidian iOS stores vaults in
`iCloud Drive/Obsidian/<vault>`, which is a third-party app container, not a
plain iCloud Drive folder. Whether iCloud for Windows surfaces app containers
under `%USERPROFILE%\iCloudDrive\` is a fact nobody here has checked.

**If it does not, the Windows bridge is impossible as specified** and the mac
becomes the only bridge — which is survivable (`D11` loses its redundancy
argument, not its correctness) but changes the deployment entirely. Check
this before E1; it is five minutes and it can invalidate more of the spec
than anything else on this list.

### E3 — do the two bridges actually converge?

Run two `xsync plan` loops — mac and Windows host — against the real trees
with apply disabled, for a day. Diff the plans. They should differ only by
timing. Anything structural is a bug in the model, not in the code.

### E4 — what is the real conflict rate?

A month of one-writer-at-a-time. Informs
[D4](10-decisions.md#d4--deletes-resolve-to-conflict-not-to-delete) and
`max_conflict_count`, and if it is genuinely zero, the conflict machinery is
untested code and should stay maximally conservative rather than be tuned.

### E5 — does normalization fire?

Create `café.md` and a Cyrillic-named note on the mac, let them reach
Windows, compare raw bytes. [04](04-platforms.md#unicode-normalization)
predicts NFD/NFC divergence and an infinite create/delete loop. Predicted, not
observed.

## Known-unknown, no experiment designed

- **Google Drive File Provider on macOS** gives no POSIX-visible dehydration
  signal ([04](04-platforms.md)). The fallback is a read timeout, which is
  weak. Nobody has looked for a better signal.
- **x1's rclone mount** reports everything materialized. Same gap, same weak
  fallback.
- **Retention sweep and `..`** — the only `remove_file` in the program
  ([03](03-safety.md#quarantine)). Specified, untested, and the single
  function most worth attacking.
- **mtime granularity** across APFS / NTFS / DrvFs / ext4 / rclone has not
  been measured. The spec assumes mtime is only ever a hint to hash
  ([02](02-model.md)); if some surface rounds hard enough to make mtime
  useless, the cheap pre-filter disappears and every scan hashes.

## Repos this touches

- `alehatsman/dotfiles` — `#81` (the source-of-truth decision xsync
  rewrites), `#97` (iPhone), `#72` (gdrive-mirror), `#39` (cloud clients),
  `#98` (work_mac). None are closed by this repo; all are affected by it.
- Deployment is provision components, never an `xsync install`
  ([01](01-scope.md)).
