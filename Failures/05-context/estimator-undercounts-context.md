---
type: failure
concepts: [token-estimation, compaction-cut-point]
harnesses: [pi]
---
**Symptom** — Kept windows and threshold checks were smaller than reality: extension/custom messages weren't counted toward the keep budget; user-attached images counted as zero; DeepSeek V4 Flash requests still overflowed after the max-tokens clamp.

**Root cause** — The char-heuristic estimator only handled some message kinds and assumed 4 chars/token, optimistic for some tokenizers.

**Fix · [[pi]]**
- `a6f720e6c` 2026-07-09 (#6326) count custom messages in compaction budget (estimate via the same projection that builds context).
- `96f0edd02` 2026-05-26 (#4983) count user image tokens (4800 chars/image).
- `27075fe07` 2026-10-06 (#10497) pi-ai `CHARS_PER_TOKEN` 4 → 3.5 (`packages/ai/src/utils/estimate.ts:15`) — coding-agent compaction still uses chars/4 (`packages/coding-agent/src/core/compaction/compaction.ts:298-349`).

**Lesson** — Estimate exactly what will be sent (same projection, every role, images) and keep the heuristic conservative; one estimator, not two.

Related: [[token-estimation]] · [[compaction-cut-point]] · [[image-normalization]] · [[output-token-cap-misbudgeted]]
