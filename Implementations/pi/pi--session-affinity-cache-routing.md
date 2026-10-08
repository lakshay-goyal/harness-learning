---
type: implementation
harness: pi
concept: session-affinity-cache-routing
commit: b30a6dd77
files: [packages/coding-agent/src/core/sdk.ts:435, packages/agent/src/agent.ts:219-220, packages/ai/src/api/openai-prompt-cache.ts:1-8, packages/ai/src/api/openai-responses.ts:159, packages/ai/src/api/openai-responses.ts:264-286, packages/ai/src/api/openai-responses.ts:334, packages/ai/src/api/openai-completions.ts:347-348, packages/ai/src/api/openai-completions.ts:776-786, packages/ai/src/api/anthropic-messages.ts:1036-1041, packages/ai/src/api/openai-codex-responses.ts:275, packages/ai/src/api/openai-codex-responses.ts:1425-1477, packages/ai/src/api/openai-codex-responses.ts:1663-1681, packages/ai/src/api/mistral-conversations.ts:341-358, packages/coding-agent/src/core/provider-attribution.ts:67-77, packages/durable/src/harness/provider.ts:6-39]
---
[[session-affinity-cache-routing]] in [[pi]].

## Mechanism
- **Key source**: the session file id — `sessionId: sessionManager.getSessionId()` passed to `Agent` (`packages/coding-agent/src/core/sdk.ts:435`), "Session identifier forwarded to providers for cache-aware backends" (`packages/agent/src/agent.ts:219-220`). Forks/new sessions → new id ([[session-fork]]). Side requests (summaries) get a fresh `uuidv7()` so they never share cache identity ([[cache-retention-control]], `compaction.ts:629-633`).
- **Retention gate**: with `cacheRetention:"none"` adapters drop the id (Completions `cacheSessionId`, `openai-completions.ts:347-348`; Responses `:159`; Codex `:275`).
- **Per-provider wire forms**:
  | provider/api | body key | headers |
  |---|---|---|
  | OpenAI Responses | `prompt_cache_key` = clamped session (`openai-responses.ts:334`) | `openai`: `session_id` + `x-client-request-id`; `openai-nosession`: only `x-client-request-id`; `openrouter`: `x-session-id` (`:264-286`) — sent whenever sessionId (no `sendSessionAffinityHeaders` gate) |
  | OpenAI Chat Completions | — | only if `sendSessionAffinityHeaders` (default: OpenRouter): `openrouter` → `x-session-id`; `openai` → `session_id` + `x-client-request-id` + `x-session-affinity`; `openai-nosession` drops `session_id` (`openai-completions.ts:776-786, 1680`) |
  | Anthropic Messages | — | if `sendSessionAffinityHeaders` (default OpenRouter; Fireworks replica routing): `x-session-id` or `x-session-affinity` (`anthropic-messages.ts:220, 1036-1041`; `99dc6fcec`) |
  | Codex (ChatGPT) | `prompt_cache_key` (`:561`) | SSE `session-id` (hyphenated) + `x-client-request-id` (`:1663-1681`); WS same, fresh UUIDv7 if none ("models reject UUIDv4", `d2f8dafb0`) |
  | Azure Responses | `prompt_cache_key` (even when none — quirk, `azure-openai-responses.ts:202`) | — |
  | Mistral | `prompt_cache_key` | `x-affinity` unless caller overrides (`mistral-conversations.ts:341-358`) |
  | OpenCode | — | `x-opencode-session`, `x-opencode-client: pi` regardless of telemetry (`provider-attribution.ts:67-77`) |
  | pi-messages (Radius) | `options.sessionId` in body; gateway does provider translation (`api/pi-messages.ts:1-10`) | — |
- **Key clamp**: `OPENAI_PROMPT_CACHE_KEY_MAX_LENGTH = 64` code points (`openai-prompt-cache.ts:1-8`; `7be75bade` #4720); Codex session-id clamp 64 (`dcfe36c79` #6653).
- **Server-side session state as cache (Codex WebSocket)**: connection cache `Map<sessionId, Map<accountId, entry>>` (`openai-codex-responses.ts:910`); idle TTL 5 min, max age 55 min under backend 60-min limit (`:866-867`); with transport `auto`/`websocket-cached`, after success store `{lastRequestBody, lastResponseId, lastResponseItems}`; next request sends only the **delta** + `previous_response_id` if body minus input is JSON-identical and new input extends `lastInput + lastResponseItems` (`:1425-1477, 1565-1580`); `store:false` kept ("continuation works via connection-scoped previous_response_id state", `:1519-1520`); any error clears continuation; `previous_response_not_found` → retry once with full context (`:344-348`); `websocket_connection_limit_reached` → retry once; transport failure → sticky SSE fallback (`:356-371`). Debug stats delta-vs-full (`:893-947`).
- Virtual-model routers: "Returning `previous` for `continuation` … keeps prompt caches and thinking signatures valid. Switching models between turns … loses the prompt cache." (`docs/virtual-models.md:84`).

## Constants
| name | value | path:line |
|---|---|---|
| `OPENAI_PROMPT_CACHE_KEY_MAX_LENGTH` | 64 | `openai-prompt-cache.ts:1` |
| `SESSION_WEBSOCKET_CACHE_TTL_MS` | 5 min | `openai-codex-responses.ts:866` |
| `SESSION_WEBSOCKET_MAX_AGE_MS` | 55 min | `openai-codex-responses.ts:867` |
| durable provider doc | `pi.provider` `{sessionId: uuidv7()}`, `fork:"initial"` | `packages/durable/src/harness/provider.ts:12-20` |

## Evolution
- `a26a9cfab` 2026-02-13: configurable transport + Codex WebSocket session caching.
- `6af10c9c7` 2026-04-23 (#3579): Responses `session_id` header optional (proxies/routes reject).
- `99dc6fcec` 2026-05-10: session affinity for Fireworks caching.
- `ce377fc4a` 2026-05-03 (#4133): WS → SSE fallback; `78c3cbe0c` 2026-05-03 (#4103) close sockets on shutdown.
- `7be75bade` 2026-05-19 (#4720): clamp OpenAI cache keys.
- `26f1e00f7` 2026-05-27 (#4967): hyphenated Codex `session-id` (proxies strip underscore headers).
- `d0e0b84cb` 2026-06-23 (#5973): reconnect on connection limit.
- `23d146261` 2026-07-03 (#6268): rotate stale sockets (60-min backend limit).
- `dcfe36c79` 2026-07-14 (#6653): clamp Codex session-id 64.
- `d2f8dafb0` 2026-07-19 (#6834): shared UUIDv7 (Codex rejects v4).
- `c5dcb2600` / merge `906b40a75` 2026-07-22 (#6955): `previous_response_not_found` → retry without continuation.
- `9b3a20591` 2026-07-22, `241431c69` 2026-07-23 (#6618): summaries get fresh routing id + no cache.
- `cfe6b6a05` / merge `784653468` 2026-07-31 (#7364, #7284): sockets scoped to account.
- `70eceaade` 2026-10-04 (durable): persisted per-conversation provider session UUIDv7.

## Evidence commits
`a26a9cfab`, `6af10c9c7`, `99dc6fcec`, `ce377fc4a`, `78c3cbe0c`, `7be75bade`, `26f1e00f7`, `d0e0b84cb`, `23d146261`, `dcfe36c79`, `d2f8dafb0`, `c5dcb2600`, `906b40a75`, `9b3a20591`, `241431c69`, `cfe6b6a05`, `784653468`, `70eceaade`.

## Quirks
- Header-name sensitivity: underscore headers (`session_id`) stripped by proxies → hyphenated variant; OpenCode Responses models omit `session-id` (CHANGELOG `:1251`, #6645).
- Inconsistent gating: Responses always sends affinity headers when a session exists; Completions/Anthropic only with `sendSessionAffinityHeaders`.
- `connectWebSocket` deletes `wsHeaders["OpenAI-Beta"]` but `Headers.entries()` yields lowercased names — delete likely a no-op (`openai-codex-responses.ts:1087-1088`; `headersToRecord` copies `headers.entries()` keys verbatim, `packages/ai/src/utils/headers.ts:3-8`, and Fetch `Headers` normalizes names to lowercase — inferred, not runtime-tested).

## Durable variant (packages/durable)
- `ProviderDoc` (`kind:"pi.provider"`, conversation scope, `fork:"initial"` → "every fork starts with a fresh identity instead of copying its parent") persists `sessionId` across reopen/reset/compaction; legacy conversations get one migration commit (`packages/durable/src/harness/provider.ts:6-39`); forwarded by generation (`harness/generation.ts:204`). Live gate `test/provider-session-cache-e2e.test.ts` (real Codex cache reuse). Fix `70eceaade` → [[cache-affinity-lost-across-reopen]].

## Failures
- [[server-limited-identifier-rejected]]
- [[connection-cache-shared-across-accounts]]
- [[persistent-connection-lifetime-exceeded]]
- [[transport-fallback-after-partial-output]]
- [[cache-affinity-lost-across-reopen]]
- [[side-request-cache-pollution]]
