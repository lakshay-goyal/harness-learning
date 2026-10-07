---
type: implementation
harness: pi
concept: cache-warming
commit: b30a6dd77
files: [packages/coding-agent/src/core/cache-warmer.ts:15-32, packages/coding-agent/src/core/cache-warmer.ts:39-70, packages/coding-agent/src/core/cache-warmer.ts:205-257, packages/coding-agent/src/core/cache-warmer.ts:284-400, packages/coding-agent/src/core/sdk.ts:367-381, packages/coding-agent/src/core/sdk.ts:420-431, packages/coding-agent/src/core/settings-manager.ts:80-82, packages/coding-agent/src/core/settings-manager.ts:1045-1049, packages/ai/scripts/generate-models.ts:979-992]
---
[[cache-warming]] in [[pi]].

## Mechanism
- **Trigger**: every session-scoped request (re)starts a warming run: `streamFn` calls `cacheWarmer.start({model, context, options}, isCurrent)` only when `options.sessionId === sessionManager.getSessionId()` — "Compaction and summaries use their own routing ids; only session requests replace the cache entry" (`packages/coding-agent/src/core/sdk.ts:420-431`).
- **Eligibility** (`cache-warmer.ts:205-229` `start`): mode ≠ off; `isReplayable` — skip Anthropic budget-based thinking (non-adaptive Claude) because `budget_tokens` derives from `max_tokens`, "which Anthropic keys the message cache on, and the model could still think for thousands of tokens" (`:48-58`); TTL from `model.promptCache[retention]` (retention: options or `PI_CACHE_RETENTION`, else short; none → ineligible) (`:39-46`); no lifetime → not eligible (`docs/models.md:93-99`).
- **isCurrent** guard: same provider/id as selected model **and** current messages extend the request's messages by object identity (`sdk.ts:367-381`) — "Warm only requests for the selected model. Requests a virtual selection routed, or that an extension redirected, may not be repeated by the next request, so warming them could be wasted." Context change → stop "conversation context changed" (`cache-warmer.ts:363-369`).
- **Timing**: refresh at `min(0.9·TTL, TTL−10s)`; TTL ≤10s → none (`:28-32`; 5 min → 270 s). Late-timer guard: `refreshDeadlineAt = nextWarmAt + floor((TTL−delay)/2)`; checked before and after the decision; missed → stop "cache refresh deadline missed" — "a late refresh is likely a full-price cache write, not a cache warm" (`:284-298, 300-303, 357-361`).
- **Horizons**: streaming max 60 min after the real request (`MAX_WARMING_AGE_MS`, `:15-16`); idle 30 min (`MAX_IDLE_WARMING_AGE_MS`, `:17-18`; `onAgentSettled` `:245-257`).
- **Modes** `cacheWarming: off | streaming | idle`, default `streaming`, **global settings only** — "Read from global settings only because warming costs money" (`settings-manager.ts:80-82, 1045-1049`; doc `docs/settings.md:21`). `streaming` stops at agent settle; `idle` keeps warming between runs.
- **Economics** `evaluate` (`:378-400`): `promptTokens` = last assistant `input+cacheRead+cacheWrite` on branch (`:61-70`); `warmCost = price(cacheRead:prompt, output:1)`; `missCost = max(0, price(cacheWrite‖input : prompt) − price(cacheRead : prompt))`; `p = 1` streaming, `0.15` idle; warm iff `p·missCost − warmCost ≥ $0.05`; `economicsAvailable=false` if prompt size or prices unknown → stop "cache economics unavailable". Uses `calculateCost` incl. tiers (`:72-86`) → [[usage-cost-accounting]].
- **Extension override**: `cache_warming_decision` event with `{warmCost, missCost, continuationProbability, action}`; result `{action}`; extension failure falls back to pi's decision (`:108-120, 307-317`; wiring `sdk.ts:334-339`).
- **Refresh request**: exact last `(model, context, options)` with `maxTokens: 1, maxRetries: 0`, own AbortController (`:331-339`); success → session `usage` entry kind `cache_warm` (provider, responseModel, usage, "extension override" note) — **never enters model context** (`:340-349`; `docs/settings.md:21`). Errors swallowed: "Cache warming is best-effort and must not affect the active agent run" (`:351-353`). Then reschedule.
- `/session` shows status/next decision (`:403-453`). `cache_warm` usage entries reset the miss baseline in [[cache-miss-accounting]] (`cache-stats.ts:120-129`).
- **TTL metadata** only for direct Anthropic `{short:300, long:3600}`; OpenAI deliberately unset ("re-evaluate it using observed expiry, replay, and billing behavior") (`generate-models.ts:979-992`); users can declare `promptCache` per model incl. via `modelOverrides` for validated proxies (`docs/models.md:92-99`).

## Constants
| name | value | path:line |
|---|---|---|
| `MAX_WARMING_AGE_MS` | 60 min | `cache-warmer.ts:16` |
| `MAX_IDLE_WARMING_AGE_MS` | 30 min | `cache-warmer.ts:18` |
| `CACHE_WARMING_MINIMUM_EXPECTED_SAVINGS` | $0.05 | `cache-warmer.ts:20` |
| `IDLE_CONTINUATION_PROBABILITY` | 0.15 ("Measured from our own usage; per-session estimates were not better than this constant") | `cache-warmer.ts:21-26` |
| refresh delay | `min(0.9·TTL, TTL−10s)` (5 min → 270 s) | `cache-warmer.ts:29-31` |
| refresh deadline | `nextWarmAt + (TTL−delay)/2` | `cache-warmer.ts:290` |
| probe | `maxTokens: 1, maxRetries: 0` | `cache-warmer.ts:335-336` |
| default mode | `streaming` (global only) | `settings-manager.ts:1048` |
| `ANTHROPIC_PROMPT_CACHE` | `{short:300, long:3600}` s | `generate-models.ts:983` |

## Evolution
- `c596d09d9` 2026-09-19 (#9668) "add prompt cache warming": warmer, `promptCache` catalog metadata, settings, `cache_warming_decision`.
- `3390bd936` 2026-09-20 "skip late cache warming refreshes": `refreshDeadlineAt` at half the pre-expiry margin (idle timers delayed by sleep/event-loop/extension decision rebuilt expired caches at full write price) → [[late-cache-warming-is-a-full-write]].
- Virtual models: warming skipped for routed/redirected requests (`sdk.ts:374-375`; feature CHANGELOG `:251`).

## Evidence commits
`c596d09d9`, `3390bd936`.

## Quirks
- Economic gate is unusually explicit: one global measured constant (0.15) instead of per-session prediction.
- Warm uses `price(cacheWrite)` as miss cost if model has write pricing, else plain input — for 1h retention the actual write is 2× input (`models.ts:1211-1217`); whether `price()` picks the 1h rate for long retention is (unverified) — `price` builds a `Usage` without `cacheWrite1h`.
- Late check happens again after the async extension decision (`:318`) — extension latency can cancel a warm.

## Failures
- [[late-cache-warming-is-a-full-write]]
