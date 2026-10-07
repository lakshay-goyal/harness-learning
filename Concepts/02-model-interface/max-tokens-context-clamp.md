---
type: concept
stage: model-interface
tier: candidate
aliases: [clampMaxTokensToContext, CONTEXT_SAFETY_TOKENS, MIN_MAX_TOKENS, context-aware-output-budget, context-aware-output-cap, adjustMaxTokensForThinking]
harnesses: [pi]
---
Derive the requested output-token cap from the context that remains:

`min(requested or model max, window − estimated input − safety margin)`

The result never goes below the provider's minimum, and when thinking shares the cap, part of it is reserved for the answer.

## Why
- Servers that reserve `max_tokens` out of the context window reject requests where `max_tokens + input > window`.
- Static defaults are wrong somewhere: too small and they truncate, too large and they are rejected, and omitting the cap lets the server's own default (e.g. 4096) truncate ([[output-token-cap-misbudgeted]]).
- Clamping without a floor runs into provider minimums, e.g. OpenAI's 16-token minimum ([[endpoint-rejects-request-field]]).
- A thinking budget can use the entire cap ([[thinking-consumes-answer-budget]]).

## Design space
- **Cap source**
  - Fixed global cap. *pi tried 32000* and reverted it in 22a9c484e.
  - Model maximum.
  - Model maximum capped near the window: 6d474f8c1.
  - Send no cap at all: 2787b601d.
  - Context-aware clamp with a 4096-token safety margin. *pi chose this:* 09f105957.
- **Input estimate**
  - chars/4.
  - chars/3.5, which is more conservative. *pi chose this:* 27075fe07.
  - Anchored on the last real usage report (see [[token-estimation]]).
- **Provider-required caps**
  - Always send one. Anthropic requires it.
  - For Bedrock Claude, send the model cap by default. Omitting it saves throughput quota reservation but truncates at 4096: a5fac1ef0 → 11c3da4f7.
- **Floor**
  - 1 overall, 16 for OpenAI Responses.

## Implementations
- [[pi--max-tokens-context-clamp|pi]] — `clampMaxTokensToContext` in `packages/ai/src/api/simple-options.ts` (`CONTEXT_SAFETY_TOKENS=4096`, floor 1), used by the simple-options base, Anthropic and Bedrock; plus the Responses floor of 16.

## Failures
- [[output-token-cap-misbudgeted]]
- [[thinking-consumes-answer-budget]]

## Related
[[thinking-level-abstraction]] · [[token-estimation]] · [[context-overflow-detection]] · [[overflow-recovery]] · [[model-catalog]]
