---
type: failure
concepts: [http-transport-hardening]
harnesses: [pi, opencode]
---
**Symptom** — Cached Codex WebSocket sessions, and their server-side continuation context, were reused across different ChatGPT accounts in the same session (#7284).

**Root cause** — The connection cache was keyed by `sessionId` only.

**Fix · [[pi]]** — `cfe6b6a05` 2026-07-31: the cache is keyed `Map<sessionId, Map<accountId, entry>>`, with `accountId` taken from the JWT claim `chatgpt_account_id` (`packages/ai/src/api/openai-codex-responses.ts:910,1627-1638`).

**Fix · [[opencode]]** `b937fe9450` 2026-01-29: the SDK instance cache was keyed by `{npm, options}`, so two providers sharing an npm package and options shared one client (wrong base URL or key); `providerID` added to the key.

**Lesson** — Connection and cache keys must include the credential identity.

Related: [[http-transport-hardening]] · [[pi--http-transport-hardening|pi]] · [[persistent-connection-lifetime-exceeded]] · [[opencode--custom-provider-registration|opencode]]
