---
type: failure
concepts: [truncated-tool-call-guard, context-overflow-detection, overflow-recovery]
harnesses: [pi]
---
**Symptom** — Responses truncated below the intended output limit (provider context ceiling, not `max_tokens`) ended the run; length-stopped messages showed no visible error; repeated ambiguous truncation was mislabeled as overflow.

**Root cause** — "Model chose to stop", "hit max_tokens" and "hit the context ceiling" were not distinguished; for Responses every `incomplete` was a length stop.

**Fix · [[pi]]**
- `f14b3594c` 2026-06-25 — visible incomplete-response error (#4290).
- `32850ef7c` 2026-08-03 — only `max_output_tokens` is a length stop for Responses (other incomplete → error, `packages/ai/src/api/openai-responses-shared.ts:779-809`); "recoverable length" (length stop below desired max) → omit failed attempt, compact, retry **once** (`packages/ai/src/utils/overflow.ts:179`; `packages/coding-agent/src/core/agent-session.ts:3002-3008`); length-stop with 0 output at ≥99% window counted as overflow (`overflow.ts:137-171`) (#7540).
- `c7c763f5c` 2026-08-17 — ambiguous repeated truncation no longer mislabeled (#8130).

**Lesson** — Distinguish "model chose to stop", "hit max_tokens" and "hit context ceiling"; only the last is a compaction trigger, and recovery is bounded to one attempt.

Related: [[truncated-tool-call-guard]] · [[context-overflow-detection]] · [[overflow-recovery]] · [[length-truncated-tool-calls-executed]] · [[silent-overflow-undetected]] · [[pi--truncated-tool-call-guard|pi]]
