---
type: failure
concepts: [token-estimation, compaction-cut-point, auto-compaction]
harnesses: [pi, codex]
---
**Symptom** — Kept windows and threshold checks were smaller than reality: extension/custom messages weren't counted toward the keep budget; user-attached images counted as zero; DeepSeek V4 Flash requests still overflowed after the max-tokens clamp.

**Root cause** — The char-heuristic estimator only handled some message kinds and assumed 4 chars/token, optimistic for some tokenizers.

**Fix · [[pi]]**
- `a6f720e6c` 2026-07-09 (#6326) count custom messages in compaction budget (estimate via the same projection that builds context).
- `96f0edd02` 2026-05-26 (#4983) count user image tokens (4800 chars/image).
- `27075fe07` 2026-10-06 (#10497) pi-ai `CHARS_PER_TOKEN` 4 → 3.5 (`packages/ai/src/utils/estimate.ts:15`) — coding-agent compaction still uses chars/4 (`packages/coding-agent/src/core/compaction/compaction.ts:298-349`).

**Fix · [[codex]]**
- (a) API `total_tokens` did not account for encrypted reasoning items preceding the assistant message → auto-compaction triggered late: `b519267d05` 2025-11-21 "Account for encrypted reasoning for auto compaction (#7113)"; today the client adds prior-turn encrypted reasoning unless the server signals `ServerReasoningIncluded` (`codex-rs/core/src/context_manager/history.rs:893-933`).
- (b) reasoning length counted bytes-as-tokens (`estimate_reasoning_length(content.len()) as i64`, issue #9287): `1fc72c647f` 2026-01-15 "Fix token estimate during compaction (#9337)" wraps in `approx_tokens_from_byte_count`.
- (c) remote compaction pre-trim estimated model-derived base instructions while the payload used session instructions: `dc7007beaa` 2026-02-04 → [[summary-estimator-payload-mismatch]].
- Images: 7,373 bytes per resized image, patch count for original detail (`codex-rs/core/src/context_manager/history.rs:1074-1095`).

**Lesson** — Estimate exactly what will be sent (same projection, same instructions, every role, images, hidden/encrypted reasoning) in consistent units, keep the heuristic conservative, and use one estimator, not two.

Related: [[token-estimation]] · [[compaction-cut-point]] · [[image-normalization]] · [[output-token-cap-misbudgeted]] · [[codex--token-estimation|codex]]
