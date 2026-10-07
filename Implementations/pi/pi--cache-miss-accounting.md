---
type: implementation
harness: pi
concept: cache-miss-accounting
commit: b30a6dd77
files: [packages/coding-agent/src/core/cache-stats.ts:4-11, packages/coding-agent/src/core/cache-stats.ts:37-90, packages/coding-agent/src/core/cache-stats.ts:104-174, packages/coding-agent/src/modes/interactive/interactive-mode.ts:3979, packages/coding-agent/src/modes/interactive/interactive-mode.ts:4168, packages/coding-agent/src/modes/interactive/interactive-mode.ts:6670, packages/coding-agent/src/core/usage-totals.ts:30-97]
---
[[cache-miss-accounting]] in [[pi]].

## Mechanism
- Post-hoc, provider-reported: scans session entries' assistant `usage` (no request-side prediction) (`packages/coding-agent/src/core/cache-stats.ts:104-142`).
- **Miss on turn N** = `min(prevPrompt, currPrompt) − cacheRead`, where prompt = `input + cacheRead + cacheWrite` (`:62, 70`) — tokens that were in the previous request's prompt but weren't read from cache.
- **Noise floor**: misses ≤ `NOISE_FLOOR_TOKENS = 1024` ignored ("cache breakpoint granularity noise", `:10-11, 71`).
- **Cost** = missed × (actual paid per-token rate from this message's `input+cacheWrite` cost incl. write premium − cache-read rate; read rate from message or model catalog) (`:73-86`) → [[usage-cost-accounting]].
- **Sticky `reportedCache`**: a zero-cache turn counts only if the segment previously reported cache activity — distinguishes a total miss on cache-read-only providers (OpenAI-style, writes unreported) from providers that never report caching (`:37-48, 63-68, 100`).
- **Resets**: `compaction` / `branch_summary` entries reset the baseline ("The context legitimately changed"); **model switches are NOT exempt** — "they re-bill the full prompt and should be counted" (`:112-119`); `modelChanged` flag recorded (`:88`). `cache_warm` usage entries become the new baseline with `reportedCache:true` (`:120-129`) → [[cache-warming]].
- **Explanation**: `idleMs` since previous request; `CACHE_TTL_MS = 5 min` used only to attribute misses to idle gaps ("Anthropic's default cache TTL is 5 minutes", `:4-8`).
- **API**: `computeCacheWaste` (session totals), `collectCacheMisses` (per-message map for rebuilding notices on resume/post-compaction), `detectCacheMiss` (live, `message_end` before persistence) (`:144-174`).
- **UI**: transcript notices behind `showCacheMissNotices` (default false, `settings-manager.ts:1074`; CHANGELOG `:1299`) (`interactive-mode.ts:3979, 4168`); `/session` waste totals (`:6670`); footer `CH` latest cache hit rate (`db594d3a5`, CHANGELOG `:1688`).
- Aggregation: usage bucketed by `provider/responseModel ?? model`; tool/summarization usage as `Tools/summaries`; `combineUsage` preserves `cacheWrite1h`/`reasoning` (`usage-totals.ts:30-52, 61-97`).

## Constants
| name | value | path:line |
|---|---|---|
| `CACHE_TTL_MS` | 5 min | `cache-stats.ts:8` |
| `NOISE_FLOOR_TOKENS` | 1024 | `cache-stats.ts:11` |
| `showCacheMissNotices` | false | `settings-manager.ts:1074` |

## Evolution
- `db594d3a5` 2026-06-05: cache hit rate in footer.
- `3f9aa5d10` 2026-07-09 (#6427) "add prompt cache miss tracking": `cache-stats.ts`, notices, setting.
- `9993c9690` 2026-07-14: price lookup via model runtime.
- `c596d09d9` 2026-09-19: `cache_warm` entries handled as baseline.
- Upstream accuracy depends on usage normalization fixes: `6044cabb1` (#2802), `87881ca68` OpenRouter semantics, `fc3cbedc6` DeepSeek, `d3ab2af96` Kimi, `6d744f02e` (#2588) Google, `c681d35d7` reasoning double count, `0be5bb6c9`/`8a7b0c03d`/`667fc3dd3` 1h writes → [[usage-cost-accounting]].

## Evidence commits
`db594d3a5`, `3f9aa5d10`, `9993c9690`, `c596d09d9`.

## Quirks
- Using `min(prev, curr)` means shrinking prompts (e.g. after context edits) are not over-counted, but a rewrite of an early prefix byte shows up as a near-total miss — which is exactly the signal for [[cache-stable-prompt-prefix]] regressions.
- 5-min idle heuristic is Anthropic-specific; OpenAI/others' TTLs differ (explanatory text only).

## Failures
- (diagnostic tool for: [[volatile-system-prompt-prefix]], [[late-tool-change-rewrites-cache]])
