---
type: failure
concepts: [file-op-tracking]
harnesses: [codex]
---
**Symptom** — A partially failed apply_patch (a move wrote the destination, then failed to remove the source) made the whole call "unknowable" and the turn diff dropped a change that really happened; the diff drifted from the workspace.

**Root cause** — Diff tracking treated a failed multi-file patch as all-or-nothing, but patch application is not transactional (files written sequentially without rollback).

**Fix · [[codex]]** — `9b6c6f7a01` 2026-05-07 "preserve exact turn diffs after partial apply_patch failures (#21518)": "a move can write the destination file before failing to remove the source. Treating the whole call as unknowable then drops a change that Codex actually knows happened" — exact deltas tracked (`codex-rs/core/src/turn_diff_tracker.rs:92-105`); `f7e8ff8e50` same day made tracking operation-backed.

**Lesson** — Derive diffs from verified operations, including partial ones.

Related: [[file-op-tracking]] · [[partial-multi-file-patch-untracked]] · [[patch-envelope-edit]] · [[codex--file-op-tracking|codex]]
