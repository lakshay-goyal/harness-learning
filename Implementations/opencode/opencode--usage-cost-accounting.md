---
type: implementation
harness: opencode
concept: usage-cost-accounting
commit: ecc4916b5a
files: [packages/opencode/src/session/session.ts:338-391, packages/opencode/src/session/llm/ai-sdk.ts:33-45, packages/core/src/session/runner/llm.ts:340]
---
[[usage-cost-accounting]] in [[opencode]].

## Mechanism

### Legacy runtime
- Per-message usage normalized in `Session.getUsage`: `adjustedInputTokens` excludes cache; cache-write tokens read from provider-specific metadata (anthropic, bedrock, venice); reasoning priced separately and not double-counted in output (`packages/opencode/src/session/session.ts:338-371`).
- Tiered pricing: `contextTokens > 200_000` → `cost.experimentalOver200K` rates from models.dev (`session.ts:384-385`).
- Copilot: billed `total_nano_aiu` parsed from raw chunks overrides computed cost (`packages/opencode/src/session/llm/ai-sdk.ts:33-45`; `session.ts:387-391`).

### v2 runtime
- Tokens normalized (non-finite → 0) into `nonCachedInputTokens`, `visibleOutputTokens`, `cacheReadInputTokens`, `cacheWriteInputTokens`, `reasoningTokens`; `Step.Ended` writes **`cost: 0`** hard-coded (`packages/core/src/session/runner/llm.ts:340`). Contradicts `packages/llm/DESIGN.md:663-683` ("aggregate run cost is unavailable rather than partial or silently zero").

## Constants
| name | value | path:line |
|---|---|---|
| long-context tier threshold | 200 000 tokens | `packages/opencode/src/session/session.ts:384` |

## Evolution
- 2025-08-22 `1f57b9a70f` count reasoning tokens; 2025-11-12 `c8bda598f5` OpenRouter cache cost double-charged, `b63b6d04c6` aliases + cached/reasoning.
- 2026-01-27 `d8e7e915e2` Venice cache-creation tokens.
- 2026-03-02 `fd6f7133c5` Bus event shared a mutable part → token values overwritten.
- 2026-03-29 `72c77d0e7b` AI SDK v6 changed Anthropic/Bedrock input semantics → double counting.
- 2026-04-04 `280eb16e77` reasoning double-counted in output.
- 2026-05-09 `c6e6bdf59f` negative stored token counts crashed decode; 2026-06-01 `ae92f3158f` Copilot token billing; 2026-09-20 `8bf288ecb1` TogetherAI streams reported no usage until SDK bump.

## Quirks / drift
- SDK upgrades silently change usage conventions; normalize at the provider boundary.

Failures: [[usage-double-counting]] · [[streamed-usage-misread]].

Contrast: [[pi--usage-cost-accounting|pi]] prices every message from a disjoint partition incl. 1 h cache writes; opencode legacy prices similarly but v2 does not price at all yet.
