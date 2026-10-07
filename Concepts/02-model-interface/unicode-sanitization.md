---
type: concept
stage: messages
tier: must-have
aliases: [sanitizeSurrogates, unicode-surrogate-sanitization, surrogate-sanitization, from_utf8_lossy]
harnesses: [pi, opencode, codex]
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
- **Type-system guarantee** (codex)
  - Lossy UTF-8 decode at the producer (`from_utf8_lossy` on process output). A Rust `String` cannot hold lone surrogates, so no boundary sanitizer is needed. ✔ codex (`codex-rs/core/src/exec.rs:809-811`)
  - Remaining hazard: valid U+2028/U+2029 breaking line-delimited framing at JavaScript boundaries ([[line-separator-breaks-jsonl-framing]]).

## Implementations
- [[pi--unicode-sanitization|pi]] — `sanitizeSurrogates` (`packages/ai/src/utils/sanitize-unicode.ts`), applied to all text in the Anthropic, Completions, Responses, Google, Bedrock, Mistral and OpenRouter-images converters.
- [[codex--unicode-sanitization|codex]] — lossy decode at the producer; the Rust string invariant replaces boundary sanitizing; the `js_repl` JSONL framing bug was fixed with byte framing.
- [[opencode--unicode-sanitization|opencode]] — same `sanitizeSurrogates` (→ U+FFFD) over all text and tool results in `ProviderTransform.normalizeMessages`.

## Failures
- [[unpaired-surrogate-breaks-json]]
- [[line-separator-breaks-jsonl-framing]]

## Related
[[unified-provider-api]] · [[cross-provider-handoff]] · [[tool-output-truncation]] · [[remote-execution-env]]
