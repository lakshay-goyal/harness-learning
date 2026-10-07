---
type: absence
harnesses: [codex]
---
# no-local-tokenizer

Token counts are estimated from bytes and server-reported usage, not a local tokenizer.

**What's missing**
- Estimator uses `approx_tokens_from_byte_count`, 4 bytes/token (`codex-rs/utils/string/src/truncate.rs:4`).

**Evidence of decision**
- `fd0673e457` 2025-10-22 "feat: local tokenizer" (tiktoken-rs) deleted `52d0ec4cd8` 2025-11-20 "Delete tiktoken-rs (#7018)"; crate `utils/tokenizer` died the same day.

**Implication**
- Estimates are language-biased and need corrections (encrypted reasoning, images); server usage is the ground truth ([[token-estimation]]).

Related: [[token-estimation]] · [[auto-compaction]] · [[Absences]]
