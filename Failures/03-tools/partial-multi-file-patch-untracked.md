---
type: failure
concepts: [patch-envelope-edit, file-op-tracking]
harnesses: [codex]
---
**Symptom** — A failed `apply_patch` (e.g. a move that wrote the destination, then failed to remove the source) left real changes on disk, but the harness treated the whole call as unknowable and dropped it from the turn diff, so the diff "can drift from the workspace" (`9b6c6f7a01` body).

**Root cause** — The patch verifies all hunks in memory but writes files sequentially without rollback; the turn-diff tracker only recorded fully successful patches.

**Fix · [[codex]]**
- `9b6c6f7a01` 2026-05-07 "preserve exact turn diffs after partial apply_patch failures (#21518)" — `ApplyPatchFailure` carries the exact committed prefix; any failed write sets `delta.exact = false` ("A failed write can still have modified the target before surfacing an error (for example by truncating before ENOSPC)") (`codex-rs/apply-patch/src/lib.rs:438-465`).
- Same day `f7e8ff8e50` "Make turn diff tracking operation backed" — the turn diff accumulates committed patch deltas per (environment, path) instead of re-reading the filesystem ([[file-op-tracking]]).
- Still not transactional: no rollback of earlier files (observed, not stated as a decision).

**Lesson** — Non-transactional multi-file edits must report exactly what landed (and flag possibly-partial writes), not succeed/fail as a unit.

Related: [[patch-envelope-edit]] · [[file-op-tracking]] · [[apply-patch-path-and-permission-hazards]] · [[codex--patch-envelope-edit|codex]]
