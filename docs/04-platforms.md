# 04 — The surfaces, and what each one lies about

Four filesystems, each with one behaviour that breaks a naive syncer. This
document is the list of those behaviours; the rest of the program is
ordinary.

## Placeholders

A cloud client can present a file that is listed, sized, dated — and not
locally present. Opening it either blocks on a download or fails. Three
states, not two:

- **materialized** — bytes on disk
- **dehydrated** — a declaration that bytes exist elsewhere
- **partial** — the vendor daemon is mid-write ([02](02-model.md#stability)
  handles this one)

### Detection, per surface

**Windows** — authoritative and cheap. `FILE_ATTRIBUTE_OFFLINE` (0x1000),
`FILE_ATTRIBUTE_RECALL_ON_DATA_ACCESS` (0x400000),
`FILE_ATTRIBUTE_RECALL_ON_OPEN` (0x40000), read from
`std::os::windows::fs::MetadataExt::file_attributes()`. No crate, no FFI, one
bitmask on metadata xsync already has. Both iCloud for Windows and Drive's
streaming mode set these. **This is the surface the feature is for.**

**macOS** — partial. iCloud dehydration is visible by name: the real file
disappears and `.<name>.icloud` appears, a small plist. Detectable with a
string compare. Google Drive's File Provider gives nothing equivalent to a
POSIX stat; its dehydrated files look normal and only a read reveals the
truth. Policy there is a read timeout, which is weaker, and macOS is not a
primary surface anyway.

**WSL / DrvFs** — nothing. `/mnt/c` does not surface Windows file attributes,
and hydration requested through it is the known-bad path. **Policy: a
`/mnt/c` path may never be declared as a cloud side.** It is a plain tree.
The Windows-host daemon owns hydration for the Windows disk; the WSL daemon
assumes what it sees is real. If that assumption is wrong, the Windows pair
is misconfigured, and that is where the error belongs.

**Linux / rclone mount** — nothing useful. Everything reports materialized;
every read is a network call. Timeout only.

### Policy

Per side, configured: `skip` (default) | `hydrate` | `fail`.

**`skip` is the default and should never fire in the notes use case**, where
both sides are pinned *available offline*. If it fires, something changed
about the pin, and the log line is the signal. `hydrate` exists for trees
that are legitimately on-demand and must be considered carefully: it is the
setting that pulls a terabyte through a metered connection because a syncer
walked a directory.

A placeholder is reported, never propagated, never deleted, and never
compared ([02](02-model.md)).

## Unicode normalization

macOS stores filenames decomposed (NFD); Windows and Linux store what you
gave them, normally composed (NFC). `café.md` written on a Mac and `café.md`
written on Windows are **different byte sequences for the same name**, and a
syncer that compares raw bytes sees a create on one side and a delete on the
other, forever, in a loop, every poll.

This fleet writes Cyrillic and Latin-with-diacritics filenames. It will fire.

**Policy:** compare paths in NFC. Store NFC in the state snapshot. Write
using the source's original bytes — never rewrite a name that is already
correct on its own filesystem.

**If two entries in one tree normalize to the same NFC key, fail the pair**
and name both. That is unresolvable without guessing, and it is rare enough
that a human should see it.

## Case

NTFS is case-insensitive and case-preserving; ext4 is case-sensitive; APFS is
usually insensitive. `Notes.md` and `notes.md` can both exist in WSL and
cannot both exist on `C:`.

**Policy:** same shape as normalization. Compare case-insensitively when
either side is an insensitive filesystem; on a collision, **fail the pair and
name both files.** Do not pick. Do not merge.

## Path length

Windows `MAX_PATH` is 260 unless opted out. Rust's std uses `\\?\` prefixed
paths for absolute paths, which lifts it, but the limit still bites through
DrvFs and through other programs reading the same tree. **Warn above 240
bytes**, do not fail — a note with a long title is a real thing and refusing
it is worse than the warning.

## Line endings, BOMs, permissions

Not touched. xsync copies bytes. A file that round-trips through two cloud
clients and three operating systems should come back identical, and anything
xsync "helpfully" normalizes is a diff it created itself.

Mode bits are explicitly out of scope ([01](01-scope.md)) — DrvFs over NTFS
cannot represent them, so syncing them would produce a permanent
non-converging diff.

## Where the daemons run

Every machine that has two trees to reconcile runs its own pair. Pairs are
independent ([02](02-model.md)); nothing coordinates them.

| host | pair | both sides cloud? |
|---|---|---|
| mac | `~/Library/Mobile Documents/…/notes` ↔ `~/Library/CloudStorage/GoogleDrive-…/My Drive/notes` | **yes — bridge** |
| main_pc, Windows host | `iCloudDrive\…\notes` ↔ `gdrive\notes` | **yes — bridge** |
| main_pc, WSL guest | `/mnt/c/Users/aleha/gdrive/notes` ↔ `~/notes` | no |
| mini_pc, WSL guest | `/mnt/c/Users/aleha/gdrive/notes` ↔ `~/notes` | no |
| x1 | `~/.local/mnt/notes` (rclone) ↔ `~/notes` | one side |

**Two hosts own the iCloud↔Drive bridge, on purpose** ([10](10-decisions.md#d11)).
Everyone else reconciles a cloud mount against a real local tree and never
touches iCloud at all.

### What the local pairs buy

`components/notes` currently makes `~/notes` a symlink into the cloud mount,
so every read and write goes through DrvFs or FUSE. On WSL that means no
inotify, slow stat, and editor plugins that poll. On x1 it means every `zk`
query is a network call into an rclone mount.

An xsync pair replaces the link with **a real directory on ext4**, reconciled
against the mount on a timer. nvim, zk, ripgrep and git all get a local
filesystem; the cloud mount is touched only by xsync, which is the only
process that expects it to be slow.

That is a change to `components/notes` — the link and its guards stop being
the mechanism — and it needs its own issue.

### mini_pc's Windows host is deliberately not a bridge

Two bridges is a decision ([D11](10-decisions.md#d11)); three is a habit. The
marginal one buys nothing — the second bridge already covers the case where
the first host is off — and each one adds a writer to the same two cloud
namespaces.

### work_mac is out of scope

`work-notes` is a separate vault on a separate Drive folder, kept apart on
purpose ([dotfiles#81](https://github.com/alehatsman/dotfiles/issues/81)).
It has no iPhone requirement and therefore no reason to touch iCloud. If it
ever gets a pair, it is a different pair with its own name, not a second
corner on this one.
