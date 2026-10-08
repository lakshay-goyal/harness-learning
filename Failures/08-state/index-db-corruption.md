---
type: failure
concepts: [sqlite-session-index]
harnesses: [codex]
---
**Symptom** — Users' SQLite state DB became corrupted (traced to an older bundled SQLite with the WAL-reset bug); for affected DBs "recovery of all data is not possible", breaking thread listing/resume metadata.

**Root cause** — A second persistent store beside the rollout files, on an engine version with a known WAL corruption bug and no integrity check at open.

**Fix · [[codex]]**
- `3691fe5b76` 2026-06-10 (#26859) — auto-backup the corrupt DB and rebuild from rollout files ("the data is reconstructable from the rollouts on disk").
- Compile-time assert: bundled SQLite ≥ 3.51.3, "bundled SQLite must include the WAL-reset corruption fix" (`codex-rs/state/src/lib.rs:7-10`).
- `3620b2caf8` 2026-09-30 (#49701) — `PRAGMA quick_check` once at open, budget 100 ms ("Limit startup validation to 100 ms.", `codex-rs/state/src/sqlite.rs:414-419`).

**Lesson** — Keep the index derivable from the append-only log so it is disposable; validate cheaply at open and rebuild instead of failing.

Related: [[sqlite-session-index]] · [[session-tree]] · [[codex--sqlite-session-index|codex]]
