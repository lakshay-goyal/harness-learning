---
type: implementation
harness: opencode
concept: token-estimation
commit: ecc4916b5a
files: [packages/core/src/util/token.ts:3-5, packages/opencode/src/session/overflow.ts:31-33, packages/opencode/src/session/session.ts:361-364, packages/opencode/src/session/compaction.ts:215-221, packages/core/src/session/compaction.ts:83, packages/core/src/session/compaction.ts:237-240]
---
[[token-estimation]] in [[opencode]].

## Mechanism
- One heuristic for both runtimes: `Token.estimate = max(0, round(len / 4))` (`packages/core/src/util/token.ts:3-5`); no tokenizer, no image constant.

### Legacy runtime — provider usage drives thresholds, estimate only for budgeting
- Threshold uses the last step's provider usage: `tokens.total || input + output + cache.read + cache.write` (`packages/opencode/src/session/overflow.ts:31-33`).
- Usage normalization: AI SDK v6 counts cached tokens inside `inputTokens`, so `input = inputTokens − cacheRead − cacheWrite` (`packages/opencode/src/session/session.ts:361-364`, `c33d9996f0`); cache write pulled from Anthropic/Vertex/Bedrock/Venice metadata.
- chars/4 over `JSON.stringify(toModelMessages(...))` only for tail budgeting (`packages/opencode/src/session/compaction.ts:215-221`) and pruning (`Token.estimate(part.state.output)`).

### v2 runtime — pure pre-request estimate
- `estimate(JSON.stringify({system, messages, tools}))` of the request about to be sent (`packages/core/src/session/compaction.ts:83`, `:237-240`); no usage anchor.

## Constants
| name | value | path:line |
|---|---|---|
| `CHARS_PER_TOKEN` | 4 | `packages/core/src/util/token.ts:3` |

## Evolution
- 2025-09-12 `983e3b2ee3` chars/4 estimate introduced with compaction fixes.
- 2026-03-27 `c33d9996f0` AI SDK v6 cache-in-input normalization → [[usage-double-counting]].

## Quirks / drift
- JSON stringification inflates the estimate with keys/escapes, and base64 images count at full string length (inference) — conservative for text, very pessimistic for media.
- Legacy and v2 measure different things (last usage vs next request), so the same session can compact at different points in the two runtimes.

Contrast: [[pi--token-estimation|pi]] anchors on the last trustworthy usage and estimates only the trailing messages, with an image constant; opencode legacy uses raw last usage, v2 a pure chars/4 estimate of the whole request.
