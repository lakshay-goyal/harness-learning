---
type: implementation
harness: pi
concept: unicode-sanitization
commit: b30a6dd77
files: [packages/ai/src/utils/sanitize-unicode.ts:1, packages/ai/test/unicode-surrogate.test.ts:1]
---
[[unicode-sanitization]] in [[pi]].

## Mechanism
- `sanitizeSurrogates(text)` removes **unpaired** UTF-16 surrogates (high without low, low without high) which "cause JSON serialization errors in many API providers"; properly paired emoji preserved (`packages/ai/src/utils/sanitize-unicode.ts:1-25`; regex `:21-25`). Test `packages/ai/test/unicode-surrogate.test.ts`.
- Applied at the **provider boundary** inside every converter rather than at producers: 9 api modules — anthropic-messages (user text, `anthropic-messages.ts:1355-1392`), openai-completions, openai-responses-shared, google-shared/generative-ai/vertex, bedrock-converse-stream, mistral-conversations, openrouter-images.
- Custom provider docs list Unicode boundaries in the recommended test matrix (`packages/coding-agent/docs/custom-provider.md:156-169`).

## Remote-env variant (packages/env, packages/chord)
- pi-env remote connection makes strings well-formed before sending: lone surrogates become U+FFFD (replace, not drop) (`packages/env/src/connection.ts:92-95`); Chord CBOR encoder rejects lone surrogates (`encoder.ts:113-114`). Transport-level, not provider-level.

## Constants
| name | value | path:line |
|---|---|---|
| policy | drop unpaired surrogates (provider path) / U+FFFD (pi-env) | packages/ai/src/utils/sanitize-unicode.ts:21-25 |

## Evolution
- 2025-10-13 `4e7a34046` surrogate sanitization added to all providers.

## Evidence commits
4e7a34046

## Quirks
- Two different policies in the repo (delete vs replace with U+FFFD) depending on layer.
- Sanitization happens per request on every replay; persisted transcripts keep the raw text.

## Failures
- [[unpaired-surrogate-breaks-json]]
