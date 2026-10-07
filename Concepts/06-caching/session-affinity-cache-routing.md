---
type: concept
stage: caching
tier: candidate
aliases: [prompt_cache_key, x-session-affinity, x-session-id, session-id, x-affinity, x-opencode-session, pi.provider.sessionId, previous_response_id, websocket-cached, promptCacheKey, setCacheKey, x-opencode-session-id, x-parent-session-id]
harnesses: [pi, opencode]
---
Send a stable per-conversation id as a cache key and/or affinity header so the provider routes consecutive requests to the replica holding the cached prefix; optionally reuse server-side conversation state to send only deltas.

## Why
- Implicit-cache providers (OpenAI, Mistral, OpenRouter, Fireworks) shard caches across replicas; without a sticky key consecutive turns land on cold replicas.
- Keys and headers are mangled by limits and middleboxes (64-char cap, proxies stripping underscore headers) ([[server-limited-identifier-rejected]]).
- The id must survive process restart/reopen/compaction, and must change for forks and side requests ([[cache-affinity-lost-across-reopen]], [[side-request-cache-pollution]]).
- Server-side session state (WebSocket `previous_response_id`) is a cache with TTL, capacity and credential identity ([[connection-cache-shared-across-accounts]], [[persistent-connection-lifetime-exceeded]]).

## Design space
- No key (provider hashes prefix only).
- **Session id as `prompt_cache_key` + provider-specific affinity headers, dropped when retention is none** (**pi chose**).
- **Fresh id for side requests** (**pi**).
- In-memory process id vs **persisted per-conversation id, fresh on fork** (**pi-durable** `pi.provider`).
- Stateful continuation (send delta + `previous_response_id` over a pooled socket) with stateless fallback (**pi** Codex WS).
- **Key field chosen per SDK package** (`promptCacheKey` vs `prompt_cache_key`) with a per-provider opt-in/opt-out `setCacheKey` (**opencode legacy**).
- **Header set split by provider ownership**: harness-hosted providers get session/request/project/client headers, third-party ones get `x-session-affinity` + `X-Session-Id`; subagents add parent-session headers (**opencode**).
- **Strip a known id prefix** to fit the 64-char key limit instead of clamping (**opencode v2**, `ses_` + 64 hex).
- Side requests keep the session id (**opencode**, no isolation).

## Implementations
- [[pi--session-affinity-cache-routing|pi]] — session file id → per-adapter key/headers; 64-char clamp; Codex WS delta continuation; durable persisted UUIDv7.
- [[opencode--session-affinity-cache-routing|opencode]] — session id as `promptCacheKey`/`prompt_cache_key` per SDK (`setCacheKey` override), affinity + parent-session headers; v2 strips `ses_` from 68-char external ids.

## Failures
- [[server-limited-identifier-rejected]]
- [[connection-cache-shared-across-accounts]]
- [[persistent-connection-lifetime-exceeded]]
- [[transport-fallback-after-partial-output]]
- [[cache-affinity-lost-across-reopen]]
- [[side-request-cache-pollution]]

## Tradeoffs
- [[prompt-cache-strategy]]

## Related
[[cache-retention-control]] · [[http-transport-hardening]] · [[session-fork]] · [[durable-execution]] · [[virtual-model-router]]
