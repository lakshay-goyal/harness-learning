---
type: absence
harnesses: [codex]
---
# no-transactional-multi-file-patch

(weak — observed, not stated as a decision) `apply_patch` verifies all hunks in memory first but writes files sequentially without rollback.

**What's missing**
- A failure mid-way leaves earlier files written and records `delta.exact = false` (`codex-rs/apply-patch/src/lib.rs:438-465`).

**Evidence of decision**
- `9b6c6f7a01` 2026-05-07 "preserve exact turn diffs after partial apply_patch failures (#21518)" — "a move can write the destination file before failing to remove the source": the harness tracks partial application rather than preventing it.

**Implication**
- Partial edits are possible on failure; turn diff tracking must handle them ([[turn-diff-drops-known-change]], [[file-op-tracking]]).

Related: [[patch-envelope-edit]] · [[file-op-tracking]] · [[apply-patch-path-and-permission-hazards]] · [[Absences]]
