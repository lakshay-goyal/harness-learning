---
type: failure
concepts: [credential-resolution, subscription-oauth-auth]
harnesses: [pi]
---
**Symptom** — Long agent runs failed partway through with auth errors when OAuth tokens (Copilot tokens live about 30 min) expired during long tool phases.

**Root cause** — The API key or token was resolved once per run, not per LLM call.

**Fix · [[pi]]**
- `1167e8445` 2025-12-19 — `getApiKey(provider)` is called per LLM request inside the loop (#223) (`packages/agent/src/agent-loop.ts:381-407`).
- HEAD: OAuth is refreshed proactively when less than `DEFAULT_OAUTH_MINIMUM_VALIDITY_MS` (5 min) of validity remains (`packages/ai/src/auth/resolve.ts:102,161-191`).

**Lesson** — Resolve credentials per request, with a minimum-validity margin, not per session or run.

Related: [[credential-resolution]] · [[subscription-oauth-auth]] · [[pi--credential-resolution|pi]] · [[oauth-refresh-token-rotation-lost]]
