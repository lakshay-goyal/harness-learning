---
type: failure
concepts: [remote-execution-env]
harnesses: [pi]
---
**Symptom** — File watching missed or broke things:
- macOS: changes made right after `fs.watch` returned were missed (FSEvents not yet live; ~2% within 100 ms);
- Windows: native watchers held directories open and blocked renaming their parents — the watcher changed filesystem semantics the agent relies on;
- Windows: watcher identity wrong — NTFS "tunneling" gives a re-created file the deleted file's creation time.

**Root cause** — Native watch APIs have startup gaps and platform side effects; timestamps aren't identity.

**Fix · [[pi]]**
- `1965a8069` 2026-10-05 — rescan 500 ms after installing watchers (`FSEVENTS_SETTLE`) (`packages/durable/src/env/node-watch.ts:47-52`; `packages/env/daemon/src/watch.rs:22-24`).
- `864777ba6` 2026-10-04 — poll by default on Windows and network/FUSE fs (`node-watch.ts:10-13`).
- `97a600395` 2026-10-05 — identify files by volume serial + file index, as libuv does (`daemon/src/sys/windows.rs`).
- Contract: coverage established when `watch()` returns; changes may be spurious but never missed while healthy; `{overflow:true}` = rescan (`packages/durable/src/env/index.ts:110-138,208-217`).

**Lesson** — Treat native watch events as hints that trigger snapshot diffs, settle after install, and fall back to polling where watchers alter semantics.

Related: [[remote-execution-env]] · [[pi--remote-execution-env|pi]]
