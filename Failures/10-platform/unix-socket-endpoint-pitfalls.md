---
type: failure
concepts: [client-server-session-split]
harnesses: [pi]
---
**Symptom** — Experimental server endpoints failed or were mis-detected:
- default `~/.pi/server/<uuid>.sock` paths exceeded the Unix socket path limit (~104 bytes on macOS) and bind failed;
- tests flaked by assuming a fresh socket never reuses a dead one's inode;
- hidden dot-prefixed temp entries (`.p-`, `.c-`, `.s-`) cluttered runtime dirs and confused discovery/cleanup.

**Root cause** — `sun_path` limits are far below `PATH_MAX`; filesystems reuse inodes; hidden temp files in a scanned directory look like endpoints.

**Fix · [[pi]]**
- `9fcaac9b1` 2026-08-14 — bounded runtime paths; caller must pick a short dir; tests moved from `tmpdir()` to `/tmp` (`packages/server/README.md:59`).
- `e36b150f7` 2026-07-31 — verify liveness by connecting instead of comparing dev/ino.
- `9ad77a2c6` 2026-08-15 — rename to `bind-`, `cleanup-`, `stale-`; remove private bind path promptly. HEAD atomic bind: private `bind-<sha256(path)[:8]>` → `lstat` → `link()` → chmod → unlink; stale sockets removed only if dead AND same inode (`packages/server/src/transports/unix/listener.ts:52-80,294-336`).

**Lesson** — Treat socket files as a tiny filesystem protocol: short paths, atomic publish, liveness by connect, no hidden temp files in discovery dirs.

Related: [[client-server-session-split]] · [[pi--client-server-session-split|pi]]
