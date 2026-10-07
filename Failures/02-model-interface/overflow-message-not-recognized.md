---
type: failure
concepts: [context-overflow-detection]
harnesses: [pi]
---
**Symptom** New backends reported context overflow in wording that no pattern matched. pi then treated it as a generic error and either got stuck repeating the request or surfaced a failure instead of compacting. Anthropic 413 `request_too_large` (byte size, e.g. images) left the session stuck repeating an oversized-image request (#2734).

**Root cause** Overflow classification is string matching over `errorMessage`. Every provider words it differently, so the catalogue is inherently open-ended.

**Fix · [[pi]]** Each fix adds a pattern to `OVERFLOW_PATTERNS` (`packages/ai/src/utils/overflow.ts:37-63`):
- `946efe4b4` 2026-01-08: underscore `context_length_exceeded` form.
- `ade6a35e7` 2026-03-08: z.ai `model_context_window_exceeded` (#1937).
- `bc8eb74b8` 2026-03-27: Ollama `prompt too long; exceeded … context length` (#2626).
- `39b1bf7b6` 2026-04-01: Anthropic `request_too_large` (#2734).
- `7c5c3d6fd` 2026-05-16: LiteLLM `exceeds … maximum context length of N tokens` (#4563).
- `fa1180b6b` 2026-05-26: Poolside / OpenRouter `maximum allowed input length` (#4943).
- `121f0edbf` 2026-06-13: parenthesized `(N)` form (#5677).
- `21cb3807e` 2026-07-02: DS4 `prompt has N tokens, but the configured context size is N` (#6262).
- `0e283203c` 2026-09-20: z.ai `Prompt too long` (#9805).
- `3dd803d7e` 2026-09-30: Z.AI CN `Prompt exceeds max length` (#10208).

The file carries a "how to add a pattern" recipe for custom providers (`overflow.ts:122-131`). Custom providers should normalize to `context_length_exceeded` (`packages/coding-agent/docs/custom-provider.md:152-154`).

**Lesson** Byte-size limits count as context overflow too. Expect a regex catalogue to grow forever, and give custom providers a normalization target.

Related: [[context-overflow-detection]] · [[rate-limit-misread-as-overflow]] · [[silent-overflow-undetected]] · [[overflow-recovery]] · [[pi--context-overflow-detection|pi]]
