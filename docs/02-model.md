# 02 — The model

Everything here is about one pair. Pairs never interact; a chain is two pairs
that happen to share a path, and each is blind to the other.

## Three observations, not two

The whole of bidirectional sync is this: **two trees are not enough
information.** If A has `note.md` and B does not, you cannot tell whether A
created it or B deleted it. You need a third thing — what both sides looked
like the last time they agreed.

So: `plan(state, scan_a, scan_b) -> Vec<Action>`.

`state` is the snapshot written after the last successful apply. It is the
only persistent thing xsync owns, and it is advisory: **losing it is never
destructive**, it degrades to a first-run union ([below](#first-run)).

## Entry

What a scan yields per relative path:

```rust
enum Presence {
    File { size: u64, mtime: SystemTime, hash: Option<Hash> },
    Placeholder,          // declared by the cloud client; bytes not local (04)
    Dir,
    Absent,
}
```

`hash` is lazy. A scan stats; it does not read. The planner asks for a hash
only when size+mtime leave it unable to decide, which is the both-sides-
changed case and nothing else. On a vault of Markdown this means almost no
reads.

`Placeholder` is not `Absent` and not `File`. That distinction is the
program ([01](01-scope.md)). It propagates nothing and deletes nothing; it
is reported.

## State

Per tracked path, what was true at the last agreement:

```rust
struct Synced { size: u64, mtime: SystemTime, hash: Hash, at: SystemTime }
```

Stored as one file per pair, rewritten atomically (temp + fsync + rename).
Not sqlite: a notes vault is thousands of entries, the whole snapshot is a
few hundred kilobytes, and a transactional store buys nothing when the only
write is "replace all of it, once per scan, after the actions already
landed." Revisit if a pair ever exceeds ~10^5 entries.

**The snapshot is written after apply, never before.** A crash mid-apply
leaves a stale state, and a stale state re-plans actions that are already
done — which are idempotent. A state written first would record agreements
that never happened, and that *is* destructive.

### changed(side, state)

```
Placeholder            -> Unknown      (never compared)
Absent                 -> Gone
size != state.size     -> Changed
mtime != state.mtime   -> Changed      (then hash to confirm; mtime alone lies)
otherwise              -> Same
```

mtime inequality is a *hint to hash*, not a verdict. Drive and iCloud both
rewrite mtimes on their own schedule, and DrvFs rounds. Treating a mtime
bump as a content change would copy the vault back and forth forever.

## The decision table

| A | B | action |
|---|---|---|
| Same | Same | nothing |
| Changed | Same | copy A → B |
| Same | Changed | copy B → A |
| Changed | Changed, hashes equal | record only, no copy |
| Changed | Changed, hashes differ | **conflict** ([03](03-safety.md#conflicts)) |
| Gone | Same | quarantine B |
| Same | Gone | quarantine A |
| Gone | Gone | drop from state |
| Gone | Changed | **conflict** — a delete never beats an edit |
| Changed | Gone | **conflict** — same |
| Unknown (placeholder) | anything | nothing; report |
| New | Absent | copy A → B |
| Absent | New | copy B → A |
| New | New, hashes equal | record only |
| New | New, hashes differ | **conflict** |

Fifteen rows, total over the input space, no default arm. That table is the
program's core and it is a pure function of three values — which is why
[06](06-testing.md) can test it exhaustively with no filesystem at all.

**`Gone` versus `Changed` resolves to conflict, not to delete.** A deletion
is a cheap thing to re-do by hand and an expensive thing to undo. This is the
one row where xsync is deliberately annoying.

### First run

No state. Every path is `New` or `Absent`. The table already covers it: the
result is the union of both trees, and anything present on both sides with
differing content becomes a conflict rather than a guess. **A first run never
deletes** — there is no `Gone` without prior state. This is also exactly what
happens after a lost snapshot, which is why losing it is safe.

## Apply

Actions are executed one at a time, each atomic, in an order that is stable
but not meaningful — there is no transaction across actions, and there does
not need to be, because the next scan re-derives the truth.

**Copy:** write to `<dest>/.xsync/tmp/<rand>`, fsync, set mtime from source,
rename into place. Same-filesystem rename is atomic on NTFS, APFS and ext4.
The file at the destination path is never observed partially written.

**Quarantine:** rename into `<side>/.xsync/trash/<YYYY-MM-DD>/<relpath>`. A
rename, not a copy — instant, and it cannot half-succeed. Cross-device
fallback copies then unlinks, in that order.

**Conflict:** both contents survive ([03](03-safety.md#conflicts)).

### Stability

A file is only eligible for copy once its `(size, mtime)` is unchanged across
two consecutive scans, or its mtime is older than `min_age` (default 5 s).
Otherwise it is skipped this round. Without this, xsync copies the first 60 %
of a file that Drive's daemon is still writing — and the result passes every
subsequent check, because it is a valid file, just the wrong one.

## Ordering within a scan

1. Verify sentinels. Fail the pair if either is missing ([03](03-safety.md))
2. Scan A, scan B
3. Apply the circuit breakers to the *plan*, before any action runs
4. Apply actions
5. Rewrite the state snapshot

Step 3 is positioned on purpose: the breakers judge the plan, not the
outcome. A pair that would delete 400 files stops having deleted none of
them.
