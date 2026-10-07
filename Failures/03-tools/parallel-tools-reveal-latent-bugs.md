---
type: failure
concepts: [parallel-tool-execution]
harnesses: [codex]
---
**Symptom** — Parallel tool calls landed off by default (`dc3c6bf62a` 2025-10-05), were enabled six weeks later (`f5d9939cda` 2025-11-18), and then needed a string of fixes: instruction injection (`4985a7a444` 2025-11-19), "fix parallel tool calls" (`d802b18716` 2025-12-16), Chat Completions parallel calls (`5f80ad6da8` 2025-12-05, `649badd102` 2026-01-05), and an approval-scope privilege bug where approving one exec approved the whole batch (`c4b771a16f` 2026-02-10). `read_file` was dropped for gpt-5-codex with a note that the same would be done "for parallel tool call" (`f3b4a26f32` 2025-10-05).

**Root cause** — Parallelism changes invariants everywhere at once: result ordering, approval keys, prompt instructions, provider shims.

**Fix · [[codex]]**
- `f2555422b9` 2025-10-07 "Simplify parallel (#4829)" — single per-turn `RwLock<()>` gate (read = parallel-safe, write = exclusive) (`e95abcdf49:codex-rs/core/src/tools/parallel.rs:46-66,207-217`); results drained in model order (`codex-rs/core/src/session/turn.rs:2460-2488`).
- Approvals keyed by call/request id (`c4b771a16f`; earlier `084236f717` 2025-07-23 call_id on patch approvals, `6a0f709cff` 2025-08-14 on MCP approvals) → [[approval-scope-too-broad-in-parallel-batch]].
- `86b1123ff6` 2026-08-14 parallel for all model prompts (per-model flag removed).

**Lesson** — Enabling parallel tool calls is a multi-month hardening project (ordering, approval scoping, prompts, provider shims), not a flag flip; stage it and audit every per-turn key.

Related: [[parallel-tool-execution]] · [[parallel-tool-results-order]] · [[approval-scope-too-broad-in-parallel-batch]] · [[codex--parallel-tool-execution|codex]]
