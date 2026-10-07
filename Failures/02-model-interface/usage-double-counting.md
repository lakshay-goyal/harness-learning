---
type: failure
concepts: [usage-cost-accounting]
harnesses: [pi, opencode]
---
**Symptom**
- Reasoning tokens were counted twice (#3581).
- Gemini costs were double-counted (#2588).
- OpenRouter cache reads were first over-reported, then under-reported.
- DeepSeek (#3880) and Kimi (#8075) cache hits were billed as normal input.

**Root cause** Each provider's usage contract differs:
- `completion_tokens` already includes `reasoning_tokens`.
- Gemini `promptTokenCount` includes cached tokens.
- Cache hits arrive in non-standard fields: DeepSeek `prompt_cache_hit_tokens`, Kimi top-level `cached_tokens`.
- OpenRouter semantics were "corrected" from a single observation.

**Fix · [[pi]]**
- `c681d35d7` 2026-04-23: stop adding reasoning to output (#3581). Contract: `reasoning` is a subset of `output` (`packages/ai/src/types.ts:440-445`).
- `6d744f02e` 2026-03-26: `input = promptTokenCount − cachedContentTokenCount` (`packages/ai/src/api/google-generative-ai.ts:232-251`) (#2588).
- `6044cabb1` 2026-04-04 subtracted `cache_write` from `cached_tokens` (#2802). `87881ca68` 2026-05-16 restored the documented semantics (cached = reads only).
- `fc3cbedc6` 2026-04-29 reads `prompt_cache_hit_tokens` (#3880). `d3ab2af96` 2026-08-16 reads top-level `cached_tokens` (#8075). Chain at `packages/ai/src/api/openai-completions.ts:1531-1533`. Input = `max(0, prompt − read − write)` (`:1546-1559`).
- Image usage: `cacheRead = cached − cacheWrite` when OpenRouter reports `cache_write_tokens` (`packages/ai/src/api/openrouter-images.ts:167-198`).

**Fix · [[opencode]]** `c8bda598f5` 2025-11-12: OpenRouter cache cost double-charged (OpenAI-style input already includes cache). `72c77d0e7b` 2026-03-29: the AI SDK v6 upgrade changed Anthropic/Bedrock input semantics → double counting. `280eb16e77` 2026-04-04: reasoning tokens counted twice in output (`packages/opencode/src/session/session.ts:338-371`).

**Lesson** Before costing, normalize every provider into one disjoint (input, cacheRead, cacheWrite, output⊇reasoning) partition, following the documented spec rather than one observation.

Related: [[usage-cost-accounting]] · [[streamed-usage-misread]] · [[usage-priced-at-wrong-rate]] · [[pi--usage-cost-accounting|pi]] · [[opencode--usage-cost-accounting|opencode]]
