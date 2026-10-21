# xsync

Keep two local directory trees converged, when one or both of them belong to
a cloud client that moves files under you.

```text
   iPhone                                            x1 · WSL · mini_pc
   Obsidian                                               nvim + zk
      │                                                       │
      ▼                                                       ▼
 ┌──────────┐                                          ┌──────────────┐
 │  iCloud  │                                          │ Google Drive │
 └────┬─────┘                                          └──────┬───────┘
      │       Apple's daemon          Google's daemon         │
      │                                                       │
      ├────────────── xsync (mac) ────────────────────────────┤
      └────────── xsync (main_pc host) ───────────────────────┤
                                                              │
                                   xsync (WSL)  /mnt/c/…/gdrive/notes ─► ~/notes
                                   xsync (x1)   rclone mount          ─► ~/notes
```

**xsync speaks no cloud API.** No OAuth, no tokens, no network. It reconciles
two paths on one machine; the vendors' own daemons carry those paths up and
down. That is the whole reason it is small enough to trust.

## The problem it exists for

Apple reaches the iPhone. Google reaches everything else. Neither reaches
both, and running both clients over one directory is how `notes (conflicted
copy).md` gets manufactured. The bridge has to be a third thing that knows it
is a bridge.

Generic syncers — unison, `rclone bisync`, Syncthing — can copy files between
two local paths. **None of them know what a file that is not really there
is.** A dehydrated placeholder reads as a normal 4 KB entry until you open
it, and then it either blocks on a download or hands you a stub. A syncer
that does not model that state will, sooner or later, either hydrate 200 GB
onto a laptop or copy a stub over the only good copy.

That is xsync's one differentiating idea: **"this file is not really here" is
a first-class state**, not an I/O error discovered at copy time.

## Status

**Spec only. No code.** Scope, model, safety, platforms, surface, testing
and a decision record are written. Nothing is built, and nothing in here has
met a filesystem.

Two documents matter most to anyone picking this up:

- **[ROADMAP.md](ROADMAP.md)** — M0 is a measurement, not a build step, and
  it can end the project in favour of `unison`. That is a real possible
  outcome and it is deliberately the cheapest thing to reach.
- **[docs/11-open.md](docs/11-open.md)** — what is unverified, which
  experiments to run in what order, and which decisions not to re-argue
  without evidence. **E2 can invalidate more of this spec than anything else
  and takes five minutes.**
