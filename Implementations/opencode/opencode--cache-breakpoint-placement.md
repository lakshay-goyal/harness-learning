---
type: implementation
harness: opencode
concept: cache-breakpoint-placement
commit: ecc4916b5a
files: [packages/opencode/src/provider/transform.ts:358-407, packages/opencode/src/provider/transform.ts:465-484, packages/opencode/src/provider/transform.ts:1338-1340, packages/llm/src/cache-policy.ts:1-111, packages/llm/src/protocols/anthropic-messages.ts:234-241, packages/llm/src/protocols/anthropic-messages.ts:506-538, packages/llm/src/protocols/utils/cache.ts:12-16, packages/llm/src/protocols/utils/bedrock-cache.ts:19]
---
[[cache-breakpoint-placement]] in [[opencode]].

## Mechanism

### Legacy runtime — `applyCaching`: first two system + last two messages
- Applied to Anthropic-family models (providerID `anthropic`/`google-vertex-anthropic`, id containing `anthropic`/`claude`, npm `@ai-sdk/anthropic`/`@ai-sdk/alibaba`), not via `@ai-sdk/gateway`, and skipped when the user configured Anthropic automatic caching (`options.cacheControl`) (`packages/opencode/src/provider/transform.ts:465-484`).
- Marks `system.slice(0, 2)` and `nonSystem.slice(-2)` (`packages/opencode/src/provider/transform.ts:358-360`) — up to 4 breakpoints at fixed positions.
- Marker written under every SDK dialect at once: `anthropic.cacheControl`, `openrouter.cacheControl`, `bedrock.cachePoint {type:"default"}`, `openaiCompatible.cache_control`, `copilot.copilot_cache_control`, `alibaba.cacheControl` (`packages/opencode/src/provider/transform.ts:362-381`).
- Level: message-level for providerID `anthropic`/Bedrock; otherwise on the last content part, skipping tool-approval parts (`packages/opencode/src/provider/transform.ts:383-403`); deep-merged into existing `providerOptions` (`087d7da14d`).
- Vercel gateway gets `gateway.caching: "auto"` instead (`packages/opencode/src/provider/transform.ts:1338-1340`).
- System is collapsed to ≤ 2 blocks so both system markers stay stable ([[opencode--cache-stable-prompt-prefix]]).

### v2 runtime — `CachePolicy` "auto"
- `LLMRequest.cache`: undefined → `"auto"` = `{tools: true, system: true, messages: "latest-user-message"}`; `"none"` = no auto placement (manual hints still flow); object form = exact (`packages/llm/src/cache-policy.ts:18-37`). Rationale in code: "Anthropic 5m-cache write is 1.25x base, read is 0.1x, so a single reuse within 5 minutes already wins"; the latest-user-message breakpoint lets "every intra-turn API call hit the prefix" (`packages/llm/src/cache-policy.ts:5-11`, `packages/llm/src/cache-policy.ts:27-29`).
- Only for routes `anthropic-messages` and `bedrock-converse` (`RESPECTS_INLINE_HINTS`, `packages/llm/src/cache-policy.ts:42`, `packages/llm/src/cache-policy.ts:100`); manual `CacheHint`s on parts never overwritten (`packages/llm/src/cache-policy.ts:50`, `packages/llm/src/cache-policy.ts:57`, `packages/llm/src/cache-policy.ts:74`). Other strategies: `latest-assistant`, `{tail: n}` (`packages/llm/src/cache-policy.ts:85-97`).
- Lowering enforces the 4-breakpoint cap, allocated tools → system → messages, excess **silently dropped** with a warning log (`packages/llm/src/protocols/anthropic-messages.ts:234-238`, `packages/llm/src/protocols/anthropic-messages.ts:511-538`); Bedrock emits positional `cachePoint` blocks with the same cap (`packages/llm/src/protocols/utils/bedrock-cache.ts:19`).
- `ttlSeconds >= 3600` → `"1h"`, else provider default 5m (`packages/llm/src/protocols/utils/cache.ts:12-16`).

## Constants
| name | value | path:line |
|---|---|---|
| legacy markers | first 2 system + last 2 non-system | `packages/opencode/src/provider/transform.ts:359-360` |
| `ANTHROPIC_BREAKPOINT_CAP` | 4 | `packages/llm/src/protocols/anthropic-messages.ts:238` |
| `BEDROCK_BREAKPOINT_CAP` | 4 | `packages/llm/src/protocols/utils/bedrock-cache.ts:19` |
| v2 default policy | auto (tools + system + latest user) | `packages/llm/src/cache-policy.ts:18-22` |

## Evolution
- 2025-06-16 `0e3458b112` "fix cache-control" (wrong option namespace); 2025-07-05 `969ad80ed2` OpenRouter-Anthropic needs its own key; 2025-07-25 `827469c725` content-level markers for non-Anthropic providers → [[cache-marker-namespace-mismatch]].
- 2025-08-01 `8f45a0e227` placement rewritten to first-2-system / last-2.
- 2025-12-15 `7d1733c752`, 2026-01-21 `0e1a8a1839` models via `@ai-sdk/anthropic` npm or Bedrock ARNs not recognized as Claude → [[capability-sniffing-misses-opaque-ids]].
- 2026-02-01 `ca5e85d6ea` Bedrock `cachePoint.type` must be `"default"`.
- 2026-05-10 `77e6c0d329` TTL + breakpoint cap, `942630eb4a` auto placement, `9b369ee815` auto by default (v2).
- 2026-08-28 `517ee736b3` filter unreplayable Bedrock reasoning before caching.

## Quirks / drift
- Legacy "last two non-system messages" during a tool loop lands on the newest assistant/tool messages, so each step writes a new breakpoint; v2 pins one at the latest user message instead.
- Legacy has no breakpoint on tools; v2 marks the last tool, but drops tools entirely when `toolChoice` is `none` (max-steps turn), changing the prefix (`packages/llm/src/protocols/anthropic-messages.ts:515-517`).

## Failures
[[cache-breakpoints-miss-stable-segments]] · [[cache-marker-namespace-mismatch]] · [[capability-sniffing-misses-opaque-ids]]

Contrast: [[pi--cache-breakpoint-placement|pi]] places system + last tool + last message per adapter; opencode legacy marks fixed positions under every SDK key at once, v2 matches pi's three-point layout but anchors messages at the latest user turn.
