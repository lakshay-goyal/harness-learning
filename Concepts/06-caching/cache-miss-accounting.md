---
type: concept
stage: cost
tier: candidate
aliases: [cache-stats, showCacheMissNotices, computeCacheWaste, detectCacheMiss, NOISE_FLOOR_TOKENS, "CH footer"]
harnesses: [pi]
---
Per-turn detection, from provider-reported usage, of prompt tokens that were already in the previous request but were re-billed instead of read from cache — with cost, idle gap and model-change attribution.

## Why
- Cache regressions are silent: the request succeeds, only the bill grows. Without per-turn accounting, prefix-instability bugs ([[volatile-system-prompt-prefix]], [[late-tool-change-rewrites-cache]]) go unnoticed.
- Providers differ in what they report (some report reads only, some nothing), so naive "cacheRead == 0 ⇒ miss" misfires.

## Design space
- Aggregate hit-rate only (footer `CH` in **pi**).
- **Per-turn miss = min(prev, curr prompt) − cacheRead, noise floor, sticky "provider reports cache" flag, resets at legitimate context changes (compaction/branch summary) but not model switches** (**pi chose**).
- Request-side prediction (diff the prefix bytes) — not in pi.
- Surface as opt-in transcript notices (**pi** `showCacheMissNotices`, default off).

## Implementations
- [[pi--cache-miss-accounting|pi]] — `cache-stats.ts`; 1024-token noise floor; 5-min idle attribution.

## Failures
- (diagnostic for [[volatile-system-prompt-prefix]], [[late-tool-change-rewrites-cache]])

## Related
[[usage-cost-accounting]] · [[cache-warming]] · [[cache-stable-prompt-prefix]] · [[auto-compaction]]
