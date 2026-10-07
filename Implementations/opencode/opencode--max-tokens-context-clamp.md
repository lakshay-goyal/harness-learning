---
type: implementation
harness: opencode
concept: max-tokens-context-clamp
commit: ecc4916b5a
files: [packages/opencode/src/provider/transform.ts:18, packages/opencode/src/provider/transform.ts:1481-1483, packages/opencode/src/effect/runtime-flags.ts:52, packages/llm/src/protocols/anthropic-messages.ts:510]
---
[[max-tokens-context-clamp]] in [[opencode]].

## Mechanism
- Legacy: `maxOutputTokens = min(model.limit.output, OUTPUT_TOKEN_MAX) || OUTPUT_TOKEN_MAX` (`packages/opencode/src/provider/transform.ts:1481-1483`). **Not clamped against remaining context**; overflow is handled by compaction instead ([[opencode--context-overflow-detection|context-overflow-detection]]).
- `OUTPUT_TOKEN_MAX = 32_000` (`transform.ts:18`), overridable via `OPENCODE_EXPERIMENTAL_OUTPUT_TOKEN_MAX` (`packages/opencode/src/effect/runtime-flags.ts:52`); also feeds the compaction reserve and the thinking-budget maximum (31 999).
- Plugins can override via `chat.params`.
- v2 Anthropic lowering: `request.model.defaults.limits.output ?? route default ?? 4096` (`packages/llm/src/protocols/anthropic-messages.ts:510`).

## Constants
| name | value | path:line |
|---|---|---|
| `OUTPUT_TOKEN_MAX` | 32 000 | `packages/opencode/src/provider/transform.ts:18` |
| v2 Anthropic fallback | 4096 | `packages/llm/src/protocols/anthropic-messages.ts:510` |

## Evolution
- 2025-07-10 `469f667774` set max output token limit to 32 000.
- 2025-12-18 `1e4bfbcf6f` env override.
- 2026-01-29 `870c38a6aa` a refactor had hardcoded `maxOutputTokens = isCodex ? undefined : undefined` → every model fell back to the provider default output limit.

## Quirks / drift
- Caps output at 32 k even for 64 k/128 k-capable models.

Failures: [[output-token-cap-misbudgeted]] · [[thinking-consumes-answer-budget]].

Contrast: [[pi--max-tokens-context-clamp|pi]] tried a fixed 32 000 cap and reverted to a context-aware clamp; opencode kept the fixed cap.
