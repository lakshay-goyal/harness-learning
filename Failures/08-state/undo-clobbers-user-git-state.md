---
type: failure
concepts: [session-fork, file-op-tracking]
harnesses: [codex]
---
**Symptom** — `/undo` restored the ghost-commit snapshot with `git restore --staged`, wiping the user's staged index (issue #8214).

**Root cause** — Workspace undo built on the user's own git repository touched state the user owns (index), not just the working-tree files the agent changed.

**Fix · [[codex]]**
- `014235f533` 2025-12-20 — remove `--staged` from the restore.
- `7a8407bbb6` 2025-12-22 "chore: un-ship undo" — two days later the feature was withdrawn; `45727b9ed3` dropped it from docs; ghost snapshots removed from the API 2026-04-27 `4e05f3053c` (#19481, "make undo a no-op that reports the feature is unavailable"). `undo` is now a `Stage::Removed` no-op flag (`codex-rs/features/src/lib.rs:425-428`, `:1006-1009`) → [[no-checkpoints-undo]].
- History: `e0fbc112c7` 2025-09-23 git tooling for undo, `e92c4f6561`/`afc4eaab8b` 2025-10-27 ghost commits + `/undo`, default-on `052b052832` 2025-11-11.

**Lesson** — Snapshot/undo built on the user's git repo must never touch the index or refs the user owns; prefer a shadow store.

Related: [[session-fork]] · [[file-op-tracking]] · [[no-checkpoints-undo]] · [[codex--session-fork|codex]]
