---
type: concept
stage: caching
tier: candidate
aliases: [CacheWarmer, cacheWarming, cache_warm, cache_warming_decision, IDLE_CONTINUATION_PROBABILITY, prewarm_with_history, "generate: false", PrewarmInput]
harnesses: [pi, codex]
---
Cost-aware keep-alive: shortly before the provider's cache TTL expires, replay the last request with a 1-token output cap so the next real turn still hits cache — only when expected savings exceed the refresh cost.

## Why
- Long tool runs, slow user replies and thinking pauses outlive short TTLs (Anthropic 5 min) → the next turn re-writes the whole prefix at write price.
- Blind keep-alives waste money (idle users rarely return in time) and a late refresh is itself a full write ([[late-cache-warming-is-a-full-write]]).
- Replay must not change the cache key (Anthropic budget-thinking keys on `max_tokens`).

## Design space
- No warming (default for most harnesses).
- Fixed-interval keep-alive pings.
- **EV-gated refresh: `p·missCost − warmCost ≥ threshold`, p measured from usage, horizons, deadline check** (**pi chose**, $0.05, p=0.15 idle / 1 streaming).
- Modes: during run only (**pi default**) vs also idle; global-only setting because it costs money (**pi**).
- Only for models with verified TTL metadata (**pi**: direct Anthropic) vs any provider.
- Plugin override of each decision (**pi** `cache_warming_decision`).
- **Pre-turn warmup instead of a TTL keep-alive**
  - A `generate=false` `response.create` over the WebSocket at turn start, thread start, or idle-thread preparation with the history prefix. It prepares the connection and a server-side response that the next request continues if its prompt extends the history and its settings match. No cost model and no TTL. ✔ codex (`codex-rs/core/src/client.rs:17-26,2176-2226`; `codex-rs/core/src/codex_thread.rs:267-284`)

## Implementations
- [[pi--cache-warming|pi]] — refresh at min(0.9·TTL, TTL−10s); deadline at half the margin; usage logged as `cache_warm`, never in context.
- [[codex--cache-warming|codex]] — variant. WebSocket `generate=false` prewarm (empty or with history); counts as the first connection attempt; also used by Guardian side-model pools.

## Failures
- [[late-cache-warming-is-a-full-write]]
- [[prewarm-blocks-turn-start]]

## Related
[[cache-retention-control]] · [[cache-miss-accounting]] · [[usage-cost-accounting]] · [[model-catalog]] · [[virtual-model-router]] · [[session-affinity-cache-routing]] · [[cache-strategy]]
