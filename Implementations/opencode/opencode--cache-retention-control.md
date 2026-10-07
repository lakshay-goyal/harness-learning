---
type: implementation
harness: opencode
concept: cache-retention-control
commit: ecc4916b5a
files: [packages/llm/src/cache-policy.ts:24-37, packages/llm/src/cache-policy.ts:44-45, packages/llm/src/protocols/utils/cache.ts:12-16, packages/llm/src/protocols/anthropic-messages.ts:240-241, packages/core/src/session/compaction.ts:201-210]
---
[[cache-retention-control]] in [[opencode]].

## Mechanism

### v2 runtime (`packages/llm`)
- Knob is part of `LLMRequest.cache`: `"none"` disables auto placement (manual hints still honoured), object form carries `ttlSeconds` (`packages/llm/src/cache-policy.ts:24-37`, `packages/llm/src/cache-policy.ts:44-45`).
- `ttlBucket`: `ttlSeconds >= 3600` → `"1h"`, else provider default 5 minutes; Anthropic and Bedrock share the mapping (`packages/llm/src/protocols/utils/cache.ts:12-16`; `EPHEMERAL_5M` / `EPHEMERAL_1H`, `packages/llm/src/protocols/anthropic-messages.ts:240-241`).
- The session runner sets no `cache` field, so every request uses the 5-minute auto policy; the compaction side request is built the same way and shares the session's affinity headers (`packages/core/src/session/compaction.ts:201-210`; `0033bb3559`) (inference: no `cache: "none"` on side requests).

### Legacy runtime
- No neutral knob: markers are always `ephemeral` (5 m); users can switch Anthropic to provider-side automatic caching via `options.cacheControl`, which disables opencode's own markers (`packages/opencode/src/provider/transform.ts:468-481`); `setCacheKey: false` opts out of cache keys ([[opencode--session-affinity-cache-routing]]).

## Constants
| name | value | path:line |
|---|---|---|
| long TTL threshold | `ttlSeconds >= 3600` → `1h` | `packages/llm/src/protocols/utils/cache.ts:15-16` |
| default retention | 5 m ephemeral | `packages/llm/src/protocols/anthropic-messages.ts:240` |

## Evolution
- 2026-05-10 `77e6c0d329` (#26779) cache hint TTL; `942630eb4a` / `9b369ee815` auto policy on by default.

## Quirks / drift
- Summaries are not cache-isolated: no `none` retention and the same session id → exposure to [[side-request-cache-pollution]] (unverified by measurement).

Contrast: [[pi--cache-retention-control|pi]] exposes none/short/long per adapter and forces `none` + a fresh id on summaries; opencode v2 has the TTL plumbing but never sets it from the session.
