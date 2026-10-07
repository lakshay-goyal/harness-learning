---
type: concept
stage: messages
tier: candidate
aliases: [sanitizeSurrogates, unicode-surrogate-sanitization, surrogate-sanitization]
harnesses: [pi, opencode]
---
Remove or replace invalid text, mainly unpaired UTF-16 surrogates, at the provider serialization boundary, so that tool output or user text cannot break JSON encoding of the request.

## Why
- Tool output, such as truncated binary-ish text or slices that cut an emoji in half, contains lone surrogates. Many provider APIs reject the JSON, or it fails to serialize, so every subsequent turn fails ([[unpaired-surrogate-breaks-json]]).

## Design space
- **Where**
  - At each producer (tools, truncation).
  - At the provider boundary in every message converter. *pi chose this:* 4e7a34046.
- **How**
  - Strip unpaired surrogates, keeping valid pairs. *pi-ai does this.*
  - Replace them with U+FFFD. *pi-env/Chord transports do this.*
  - Reject. *The Chord encoder rejects lone surrogates.*

## Implementations
- [[pi--unicode-sanitization|pi]] — `sanitizeSurrogates` (`packages/ai/src/utils/sanitize-unicode.ts`), applied to all text in the Anthropic, Completions, Responses, Google, Bedrock, Mistral and OpenRouter-images converters.
- [[opencode--unicode-sanitization|opencode]] — same `sanitizeSurrogates` (→ U+FFFD) over all text and tool results in `ProviderTransform.normalizeMessages`.

## Failures
- [[unpaired-surrogate-breaks-json]]

## Related
[[unified-provider-api]] · [[cross-provider-handoff]] · [[tool-output-truncation]] · [[remote-execution-env]]
