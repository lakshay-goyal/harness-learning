---
type: failure
concepts: [max-tokens-context-clamp]
harnesses: [pi]
---
**Symptom** The default output cap went wrong in turn, across 5+ commits:
- Every model was clamped to 32000 output tokens (#4539).
- Models whose maxTokens ≈ the context window produced impossible requests (#4614).
- vLLM-style servers reserving `max_tokens` from the window rejected requests (#4675).
- Servers counting input+output against one window rejected long requests (#5595/#6061).
- The token estimate was too optimistic, so DeepSeek V4 Flash still overflowed (#10497).
- Bedrock omitted maxTokens, so its 4096 default truncated Claude output (#4848).
- A clamped value below 16 was rejected by OpenAI Responses (#6265).

**Root cause** A static default `max_tokens` is wrong for some backend. The output budget must be derived from the remaining context, using a conservative estimator.

**Fix · [[pi]]**
- `22a9c484e` 2026-05-16: respect the model max (#4539).
- `6d474f8c1` 2026-05-17: cap at 32000 when within 1024 of the context window (#4614).
- `2787b601d` 2026-05-19: stop sending default caps, keeping the required Anthropic `max_tokens` (#4675).
- `09f105957` 2026-06-25: `clampMaxTokensToContext = min(maxTokens, contextWindow − estimate − 4096)`, floor 1 (`packages/ai/src/api/simple-options.ts:15-22,46`; Anthropic `anthropic-messages.ts:968`; Bedrock `bedrock-converse-stream.ts:562`) (#5595/#6061).
- `27075fe07` 2026-10-06: `CHARS_PER_TOKEN` 4 → 3.5 (`packages/ai/src/utils/estimate.ts:15`) (#10497).
- `a5fac1ef0` 2026-04-19 omitted Bedrock maxTokens to save TPM quota. `11c3da4f7` 2026-05-21 sends the model cap by default for Claude (`bedrock-converse-stream.ts:263`) (#4848).
- `2e4ad6a09` 2026-07-06: floor of 16 (see [[endpoint-rejects-request-field]]).
- `8973ae28a` 2026-07-09: estimate ignores usage older than a later-inserted prefix message (compaction summary) (`estimate.ts:71-95`) (#6464).

**Lesson** Derive the output budget from remaining context, using a conservative token estimator, a safety margin and provider minimums. Default to the model cap for agents rather than the server default.

Related: [[max-tokens-context-clamp]] · [[token-estimation]] · [[thinking-consumes-answer-budget]] · [[context-overflow-detection]] · [[pi--max-tokens-context-clamp|pi]]
