---
type: implementation
harness: opencode
concept: session-affinity-cache-routing
commit: ecc4916b5a
files: [packages/opencode/src/provider/transform.ts:1323-1336, packages/opencode/src/provider/transform.ts:1380-1384, packages/opencode/src/session/llm/request.ts:186-206, packages/core/src/session/runner/llm.ts:204-216]
---
[[session-affinity-cache-routing]] in [[opencode]].

## Mechanism

### Legacy runtime
- Cache key = session id unless `setCacheKey: false` (`packages/opencode/src/provider/transform.ts:1323-1336`): `prompt_cache_key` for `@ai-sdk/deepinfra`/`@ai-sdk/cerebras`; `promptCacheKey` for `@ai-sdk/openai`, `azure`, `xai`, `mistral`, `venice-ai-sdk-provider`, or any provider with `setCacheKey: true`. opencode-hosted GPT-5 models also get `promptCacheKey` + encrypted reasoning include (`packages/opencode/src/provider/transform.ts:1380-1384`).
- Headers (`packages/opencode/src/session/llm/request.ts:186-206`): always `x-opencode-session-id` (+ `x-opencode-parent-session-id` for subagents); opencode-hosted providers get `x-opencode-project`, `x-opencode-session`, `x-opencode-request` (user message id), `x-opencode-client`; all others get `x-session-affinity` + `X-Session-Id`; subagents add `x-parent-session-id`.

### v2 runtime
- Same header set from the runner (`packages/core/src/session/runner/llm.ts:207-214`); `providerOptions.openai.promptCacheKey` = session id with the `ses_` prefix stripped when the id is `ses_` + 64 hex (external hashed ids), keeping it within OpenAI's 64-char limit (`packages/core/src/session/runner/llm.ts:204`, `packages/core/src/session/runner/llm.ts:216`).

## Constants
| name | value | path:line |
|---|---|---|
| cache key | session id (`ses_` stripped for 68-char ids, v2) | `packages/core/src/session/runner/llm.ts:204` |
| opt-out | `setCacheKey: false` | `packages/opencode/src/provider/transform.ts:1323` |

## Evolution
- 2025-11-16 `cf266f6162` key set for any npm containing "openai" → narrowed (OpenAI-compatible endpoints rejected it) → [[endpoint-rejects-request-field]].
- 2026-02-04 `173804c097` session affinity header for Zen; 2026-04-02 `e89527c9f0` `x-session-affinity` + `x-parent-session-id`.
- 2026-04-16 `cb18f2ef40` Azure on by default.
- 2026-06-05 `7c6adcf60f` v2 omitted the key → scoped by session; 2026-06-06 `747b8daafc` bound key length → [[server-limited-identifier-rejected]].
- 2026-07-08 `ccb6b7c3ea` xAI; 2026-07-22 `542ba88602` per-SDK key selection, `fada1a538f` Mistral serialization.
- 2026-08-18 `0033bb3559` restore v2 request headers; 2026-09-30 `e9f8a210b9` namespaced session identity headers.

## Quirks / drift
- Side requests (compaction, titles) reuse the session id as key/affinity (inference) — no fresh-id isolation like pi.
- Fork creates a new session id, so a fork starts cold on implicit-cache providers.

Contrast: [[pi--session-affinity-cache-routing|pi]] clamps keys to 64 chars, drops them when retention is none and uses fresh ids for side requests; opencode keys everything by session id and splits headers by hosted vs third-party provider.
