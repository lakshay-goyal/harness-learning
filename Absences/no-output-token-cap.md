---
type: absence
harnesses: [codex]
---
# no-output-token-cap

Requests carry no `max_output_tokens`; an attempt to read it from config was reverted.

**What's missing**
- `ResponsesApiRequest` fields have no output-token cap (`codex-rs/codex-api/src/common.rs:279-304`).

**Evidence of decision**
- Reverted pair `c9e149fd5c` / `bce030ddb5` 2025-11-21.

**Implication**
- No [[max-tokens-context-clamp]]; headroom handled by `effective_context_window_percent` (95) and auto-compaction thresholds ([[auto-compaction]]).
- `response.incomplete` with other reasons (e.g. max_output_tokens) is treated as a retryable stream error (`codex-rs/codex-api/src/sse/responses.rs:418-454`).

Related: [[max-tokens-context-clamp]] · [[auto-compaction]] · [[token-estimation]] · [[Absences]]
