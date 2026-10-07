---
type: failure
concepts: [unicode-sanitization]
harnesses: [pi]
---
**Symptom** Requests failed with JSON serialization errors at the provider, especially on Anthropic, when tool output or user text contained lone UTF-16 surrogates (e.g. a truncated emoji).

**Root cause** Unpaired high or low surrogates "cause JSON serialization errors in many API providers". Text is sliced and truncated by producers, such as tool output truncation, without respecting surrogate pairs.

**Fix · [[pi]]** `4e7a34046` 2025-10-13: `sanitizeSurrogates` removes unpaired surrogates and preserves paired emoji (`packages/ai/src/utils/sanitize-unicode.ts:1-25`). It is applied in every provider converter: anthropic-messages, openai-completions, openai-responses-shared, google-*, bedrock, mistral and openrouter-images. Test: `packages/ai/test/unicode-surrogate.test.ts`.

**Lesson** Sanitize text at the provider boundary, where it is serialized, not at every producer.

Related: [[unicode-sanitization]] · [[unified-provider-api]] · [[pi--unicode-sanitization|pi]]
