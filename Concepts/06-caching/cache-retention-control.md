---
type: concept
stage: caching
tier: must-have
aliases: [cacheRetention, PI_CACHE_RETENTION, "cacheRetention: none", prompt_cache_retention, prompt_cache_options, "ttl: 1h", promptCache, ttlSeconds, ttlBucket, "cache: none"]
harnesses: [pi, opencode]
---
A provider-neutral cache retention knob (none / short / long) mapped onto each provider's TTL fields, with one-off side requests (summaries) forced to write nothing.

## Why
- Every provider spells retention differently (Anthropic `ttl:"1h"`, OpenAI `prompt_cache_retention:"24h"` or `prompt_cache_options.ttl`, Bedrock `CacheTTL.ONE_HOUR`); a harness needs one knob.
- Long retention costs more to write (Anthropic 1h writes = 2× input) — wasteful for requests never repeated.
- One-off summaries that write cache under the session's identity pay write premiums and displace the main session's cache/affinity ([[side-request-cache-pollution]]).

## Design space
- Always-default provider caching (no knob).
- **Neutral enum + env override, default short** (**pi chose**).
- **`none` for side requests + fresh routing id** (**pi**, compaction/branch summaries).
- Explicit-mode opt-out of implicit writes where the provider supports it (**pi** OpenAI GPT-5.6+ `{mode:"explicit"}`).
- Declared cache lifetimes per model/tier as catalog metadata (feeds [[cache-warming]]), only where expiry behavior is verified (**pi**: direct Anthropic only).
- Cache-friendly side requests that *reuse* the cached prefix (pi tried "cache-friendly compaction primitives", reverted next day).
- codex: absent. `ResponsesApiRequest` has no retention or TTL field (`codex-rs/codex-api/src/common.rs:279-304`; `git grep prompt_cache_retention` at `622e9e3696` → no hits). Side requests (compaction, subagents, ephemeral forks) deliberately *share* the session cache key instead of opting out ([[codex--session-affinity-cache-routing|codex]]).
- **Retention as part of the cache policy object**: `ttlSeconds` bucketed to 5m / 1h (≥ 3600 s), `"none"` disables auto placement but keeps manual hints (opencode v2 `packages/llm`); the session never sets it, so everything is 5m and side requests are not isolated (opencode).

## Implementations
- [[pi--cache-retention-control|pi]] — `CacheRetention` in pi-ai; per-adapter mapping table; summaries `cacheRetention:"none"` + `uuidv7()`.
- [[opencode--cache-retention-control|opencode]] — v2 `CachePolicy` `ttlSeconds` → `1h` bucket, `"none"` keeps manual hints; legacy has no knob (always ephemeral 5m, or provider automatic caching via `options.cacheControl`).

## Failures
- [[side-request-cache-pollution]]
- [[server-limited-identifier-rejected]]

## Related
[[session-affinity-cache-routing]] · [[cache-warming]] · [[cache-breakpoint-placement]] · [[usage-cost-accounting]] · [[auto-compaction]] · [[branch-summary]] · [[prompt-cache-strategy]]
