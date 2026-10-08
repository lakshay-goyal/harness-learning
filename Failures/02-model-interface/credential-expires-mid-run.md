---
type: failure
concepts: [credential-resolution, subscription-oauth-auth]
harnesses: [pi, opencode]
---
**Symptom** — Long agent runs failed partway through with auth errors when OAuth tokens (Copilot tokens live about 30 min) expired during long tool phases.

**Root cause** — The API key or token was resolved once per run, not per LLM call.

**Fix · [[pi]]**
- `1167e8445` 2025-12-19 — `getApiKey(provider)` is called per LLM request inside the loop (#223) (`packages/agent/src/agent-loop.ts:381-407`).
- HEAD: OAuth is refreshed proactively when less than `DEFAULT_OAUTH_MINIMUM_VALIDITY_MS` (5 min) of validity remains (`packages/ai/src/auth/resolve.ts:102,161-191`).

**Fix · [[opencode]]** `00d6841f84` 2026-04-01: console token refreshed only after expiry → 5-min eager threshold (`packages/opencode/src/account/account.ts:138`). `ef979ccfa8` 2026-02-16: GitLab mid-session token refresh. xAI refreshes 120 s early "so a single long-running tool call doesn't have to recover from a mid-flight 401" (`packages/opencode/src/plugin/xai.ts:28-30`).

**Lesson** — Resolve credentials per request, with a minimum-validity margin, not per session or run.

Related: [[credential-resolution]] · [[subscription-oauth-auth]] · [[pi--credential-resolution|pi]] · [[oauth-refresh-token-rotation-lost]] · [[opencode--subscription-oauth-auth|opencode]]
