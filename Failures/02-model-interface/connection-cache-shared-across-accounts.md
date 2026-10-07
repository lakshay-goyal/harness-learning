---
type: failure
concepts: [http-transport-hardening]
harnesses: [pi]
---
**Symptom** — Cached Codex WebSocket sessions, and their server-side continuation context, were reused across different ChatGPT accounts in the same session (#7284).

**Root cause** — The connection cache was keyed by `sessionId` only.

**Fix · [[pi]]** — `cfe6b6a05` 2026-07-31: the cache is keyed `Map<sessionId, Map<accountId, entry>>`, with `accountId` taken from the JWT claim `chatgpt_account_id` (`packages/ai/src/api/openai-codex-responses.ts:910,1627-1638`).

**Lesson** — Connection and cache keys must include the credential identity.

Related: [[http-transport-hardening]] · [[pi--http-transport-hardening|pi]] · [[persistent-connection-lifetime-exceeded]]
