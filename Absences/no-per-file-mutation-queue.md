---
type: absence
harnesses: [codex]
---
# no-per-file-mutation-queue

Concurrency control for tools is a single per-turn RwLock (parallel-safe vs exclusive); no per-file queue — parallel `exec_command`s may write the same files concurrently.

**What's missing**
- `e95abcdf49:codex-rs/core/src/tools/parallel.rs:207-217`.

**Evidence of decision**
- `is_mutating` gating removed 2026-05-12 `862b2122ee` ("That second hook no longer carried its weight").
- Per-model `supports_parallel_tool_calls` removed `86b1123ff6` 2026-08-14: parallelism always requested, constrained per tool by the harness lock.

**Implication**
- Contrast pi's [[per-file-mutation-queue]]; codex relies on tool-level exclusivity (e.g. `apply_patch` exclusive) and the model's judgment.

Related: [[per-file-mutation-queue]] · [[parallel-tool-execution]] · [[parallel-tools-reveal-latent-bugs]] · [[Absences]]
