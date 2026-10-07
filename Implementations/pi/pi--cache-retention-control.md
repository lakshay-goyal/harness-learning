---
type: implementation
harness: pi
concept: cache-retention-control
commit: b30a6dd77
files: [packages/ai/src/types.ts:116, packages/ai/src/types.ts:220-230, packages/ai/src/api/anthropic-messages.ts:65-93, packages/ai/src/api/openai-responses.ts:97-114, packages/ai/src/api/openai-responses.ts:328-346, packages/ai/src/api/openai-codex-responses.ts:275, packages/ai/scripts/generate-models.ts:979-992, packages/coding-agent/src/core/compaction/compaction.ts:619-639, packages/durable/src/harness/compaction.ts:168, packages/ai/src/models.ts:1211-1217]
---
[[cache-retention-control]] in [[pi]].

## Mechanism
- Neutral knob `CacheRetention = "none" | "short" | "long"`, default `"short"` (`packages/ai/src/types.ts:116, 220-224`); `sessionId` alongside for cache keys/routing (`:225-230`).
- Resolution: explicit `options.cacheRetention` → env `PI_CACHE_RETENTION=long` → `"short"` (`anthropic-messages.ts:65-77`; same in `cache-warmer.ts:39-42`); pi-messages: explicit, else legacy env → `long`, else undefined = backend default (`api/pi-messages.ts:347-353`). Coding-agent sets no other retention source (settings key not found; open question findings 02).
- Per-provider mapping:
  | provider | none | short | long |
  |---|---|---|---|
  | Anthropic Messages | no `cache_control` | `{type:"ephemeral"}` (5 min) | `ttl:"1h"` iff `supportsLongCacheRetention` (`:88`) |
  | OpenAI-compat (anthropic format) | no markers | ephemeral | `ttl:"1h"` iff supported (`openai-completions.ts:1077-1087`) |
  | Bedrock | no cachePoint | `cachePoint` default | `ttl: ONE_HOUR`, not compat-gated |
  | OpenAI Responses | `prompt_cache_key` omitted; explicit-mode models get `prompt_cache_options:{mode:"explicit"}` = **no implicit writes** | key only | `prompt_cache_retention:"24h"` (non-explicit) or `prompt_cache_options:{ttl:"30m"}` (GPT-5.6+ explicit mode) (`openai-responses.ts:97-114, 334-336`) |
  | Codex Responses | `prompt_cache_key` omitted (`openai-codex-responses.ts:275, 561`) | key = session | key = session |
  | Mistral | no key/`x-affinity` | key + `x-affinity` (`mistral-conversations.ts:341-358`) | same |
  | Google | ignored | ignored | ignored |
- `supportsExplicitPromptCacheMode` only for `openai` Responses models with `cost.cacheWrite > 0` — "OpenAI charges prompt-cache writes starting with the GPT-5.6 family, and exactly those models accept `prompt_cache_options`" (`generate-models.ts:967-980`).
- Sign in with ChatGPT (provider `openai`, baseUrl `https://api.openai.com/v1`, key not `sk-`) omits `prompt_cache_retention`, `prompt_cache_options`, `max_output_tokens`, `temperature` (token sharing rejects them) (`openai-responses.ts:40-47, 328-346`).
- **One-off side requests never write cache**: `completeSummarization` forces `cacheRetention:"none"` and `sessionId: options.sessionId ?? uuidv7()` — "Avoid cache writes for one-off summaries. … callers without a session ID, including branch summaries, receive a fresh routing ID" (`compaction/compaction.ts:619-639`); AgentSession passes `sessionId: undefined` (`agent-session.ts:2739`). Durable compaction same (`packages/durable/src/harness/compaction.ts:168`). Consequence: compaction itself never writes cache, and each compaction invalidates the main prefix anyway (summary replaces history).
- **TTL metadata for warming**: `ANTHROPIC_PROMPT_CACHE = {short:300, long:3600}` only for direct `anthropic` "so cache warming does not assume equivalent behavior through proxies"; OpenAI lifetimes deliberately absent — "a documented TTL alone does not establish full cache loss" (`generate-models.ts:979-992`) → [[cache-warming]].
- **Price of long retention**: 1h writes priced 2× base input, short at `cacheWrite` rate (`models.ts:1211-1217`; `0be5bb6c9` #5738) → [[usage-cost-accounting]].

## Constants
| name | value | path:line |
|---|---|---|
| default retention | `"short"` | `anthropic-messages.ts:76`; `cache-warmer.ts:42` |
| env override | `PI_CACHE_RETENTION=long` | `anthropic-messages.ts:73`; `docs/environment-variables.md:87` |
| Anthropic long TTL | `"1h"` | `anthropic-messages.ts:88` |
| OpenAI long (pre-5.6) | `prompt_cache_retention:"24h"` | `openai-responses.ts:101-103` |
| OpenAI long (5.6+) | `prompt_cache_options:{ttl:"30m"}` | `openai-responses.ts:112` |
| OpenAI none (5.6+) | `{mode:"explicit"}` | `openai-responses.ts:111` |
| `ANTHROPIC_PROMPT_CACHE` | `{short:300, long:3600}` s | `generate-models.ts:983` |
| summaries | `cacheRetention:"none"` | `compaction.ts:631` |
| 1h write price | 2× input | `models.ts:1211-1217` |

## Evolution
- `1b6a14757` 2026-01-29: `PI_CACHE_RETENTION` env for extended caching.
- `abfd04b5c` 2026-02-01: `cacheRetention` stream option.
- `4cd4cfd98` 2026-04-23: long-retention compat flag.
- `6af10c9c7` 2026-04-23 (#3579): Responses `session_id` header optional; some routes reject `prompt_cache_retention`.
- `651d10d90` 2026-06-18: Mistral prompt caching.
- `0be5bb6c9` 2026-06-15 (#5738): 1h writes at 2× input; `8a7b0c03d` (#9457) Bedrock; `667fc3dd3` (#9210) Vercel.
- `9b3a20591` 2026-07-22: "isolate summarization requests" — fresh routing session ids, caching disabled.
- `241431c69` 2026-07-23 (#6618): "don't cache write compaction or branch summaries" — `{mode:"explicit"}` for explicit-mode models; Codex key omitted on none (`packages/ai/CHANGELOG.md:513`; coding-agent CHANGELOG `:1068`).
- `cff1cf52c` 2026-08-18 "cache-friendly compaction primitives" (summarize via cached provider-context prefix) → reverted next day `8dab70281` (reason unverified).
- `c596d09d9` 2026-09-19 (#9668): `promptCache` lifetimes in catalog for warming.

## Evidence commits
`1b6a14757`, `abfd04b5c`, `4cd4cfd98`, `6af10c9c7`, `651d10d90`, `0be5bb6c9`, `8a7b0c03d`, `667fc3dd3`, `9b3a20591`, `241431c69`, `cff1cf52c`, `8dab70281`, `c596d09d9`.

## Quirks
- Azure Responses sends `prompt_cache_key` even with `cacheRetention:"none"` (`azure-openai-responses.ts:202`) — contradicts #6618 semantics elsewhere (deliberate? unverified).
- Responses adapter nulls `sessionId` on none (`openai-responses.ts:159`) so affinity headers vanish too.
- No settings.json key for retention — env var only (unverified; not found in `sdk.ts buildRequestOptions`).
- Summaries reuse caller routing id if one is passed; coding-agent deliberately passes none.

## Durable variant (packages/durable)
- Summary request pinned with `cacheRetention:"none"` (`packages/durable/src/harness/compaction.ts:168`); main generations use the persisted per-conversation `pi.provider.sessionId` ([[session-affinity-cache-routing]]).

## Failures
- [[side-request-cache-pollution]]
- [[server-limited-identifier-rejected]]
