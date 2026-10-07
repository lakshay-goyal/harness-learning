---
type: concept
stage: caching
tier: candidate
aliases: [CacheWarmer, cacheWarming, cache_warm, cache_warming_decision, IDLE_CONTINUATION_PROBABILITY]
harnesses: [pi]
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

## Implementations
- [[pi--cache-warming|pi]] — refresh at min(0.9·TTL, TTL−10s); deadline at half the margin; usage logged as `cache_warm`, never in context.

## Failures
- [[late-cache-warming-is-a-full-write]]

## Tradeoffs
- [[prompt-cache-strategy]]

## Related
[[cache-retention-control]] · [[cache-miss-accounting]] · [[usage-cost-accounting]] · [[model-catalog]] · [[virtual-model-router]]
