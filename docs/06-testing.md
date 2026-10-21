# 06 — Testing

The architectural decision that makes this testable is in
[02](02-model.md): **the planner is a pure function** of
`(state, scan_a, scan_b)`, and the executor is the only thing that touches a
filesystem. Everything hard to get right is in the pure half.

## The table, exhaustively

[02](02-model.md#the-decision-table) has fifteen rows over a finite input
space. Every row gets a unit test with no I/O, no temp dir and no clock.
Property: the function is total — there is no default arm, and a new
`Presence` variant is a compile error rather than a silent fallthrough.

## Convergence, by property test

Generate a random sequence of operations against two in-memory trees
(create, edit, delete, rename, dehydrate, remount), run the planner and a
simulated executor to fixpoint, and assert:

1. **Convergence** — after a run with no concurrent mutation, both trees are
   equal, or every inequality is a declared conflict or placeholder
2. **Durability** — every byte sequence ever written is reachable in one of
   the two trees or in a quarantine. *Nothing ever disappears.* This is the
   property the whole program exists to hold
3. **Idempotence** — a second run over an unchanged pair emits zero actions

Property 2 is the one that justifies the cost of the harness.

## The adversarial cases, written by hand

Property tests find the general; these are the specific ones already known
to be sharp, each a named test:

- state snapshot deleted mid-life → first-run union, **zero deletes**
- state snapshot stale by one apply → re-plans applied actions, idempotent
- one side empties entirely → sentinel absent → pair fails, no actions
- one side empties, sentinel somehow present → breaker trips, no actions
- `café.md` NFD on one side, NFC on the other → one file, no loop
- `Notes.md` + `notes.md` on the case-sensitive side → pair fails, both named
- file growing during the scan → skipped until stable, never truncated
- dehydrated on A, materialized on B → no copy, no delete, one report
- both sides edited, identical content → recorded, no copy, no conflict
- delete on A, edit on B → conflict, **not** a delete
- trash sweep with a crafted path containing `..` → refused

## Filesystem fake

The executor takes a trait. Tests get an in-memory implementation that can be
told to fail a rename, report a placeholder, round an mtime to two seconds,
or be case-insensitive — because reproducing those four behaviours with real
temp directories means owning four machines.

Integration tests against real temp dirs exist too, and are few: they prove
the trait's real implementation matches the fake, not that the logic is
right.

## What cannot be tested here

Whether iCloud for Windows actually sets `FILE_ATTRIBUTE_RECALL_ON_DATA_ACCESS`
on the files this fleet has. That is a measurement on the real machine, it is
the M0 gate in [ROADMAP.md](../ROADMAP.md), and no amount of unit testing
substitutes for running one `GetFileAttributesW` against a real vault.
