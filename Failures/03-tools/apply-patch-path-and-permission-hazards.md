---
type: failure
concepts: [patch-envelope-edit, tool-call-gate, path-normalization]
harnesses: [codex]
---
**Symptom** — The patch tool's permission handling widened sandbox access: deriving write permissions from the parent of an already-writable target granted more than the patch needed (`530c1aed58`), and redundant write roots were added for patches (`307e427a9b`). Siblings of the same class: aliased paths in one patch ([[duplicate-path-ops-in-one-patch]]) and partial multi-file writes the harness lost track of ([[partial-multi-file-patch-untracked]]).

**Root cause** — Patch approval/sandboxing computes a writable-path set from the patch's targets; parent-directory derivation and duplicated roots made that set larger than the edit.

**Fix · [[codex]]**
- `307e427a9b` 2026-03-27 "don't include redundant write roots in apply_patch (#16030)".
- `530c1aed58` 2026-08-20 "Prevent `apply_patch` from widening write permissions (#39614)".
- Safety path `assess_patch_safety`: empty patch rejected; patches confined to writable roots auto-approved **but still executed sandboxed** because "paths in the patch are hard links to files outside the writable roots" (`codex-rs/core/src/safety.rs:67-125`); permission preparation errors surface as "failed to prepare patch permissions" / "failed to check patch permissions" (`codex-rs/core/src/tools/handlers/apply_patch.rs:511,516`).
- `a1c88e865d` 2026-08-10 duplicate resolved paths rejected; `9b6c6f7a01` 2026-05-07 exact partial-failure deltas.

**Lesson** — A patch format is a mini-transaction with its own permission surface: canonicalize paths, grant exactly the targets (never their parents), reject aliasing, and report partial effects exactly.

Related: [[patch-envelope-edit]] · [[tool-call-gate]] · [[path-normalization]] · [[os-level-sandbox]] · [[sandbox-escalation-retry]] · [[concurrent-file-mutation-interleave]] · [[tool-result-misreports-facts]] · [[codex--patch-envelope-edit|codex]]
