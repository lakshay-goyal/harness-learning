---
type: failure
concepts: [credential-resolution]
harnesses: [pi]
---
**Symptom** — After an OAuth request, `ANTHROPIC_API_KEY` vanished for the rest of the process, so later API-key requests failed.

**Root cause** — To stop the Anthropic SDK from picking up the env key alongside OAuth, pi deleted `process.env.ANTHROPIC_API_KEY`, mutating process-global state for a per-request decision.

**Fix · [[pi]]**
- `a248e2547` 2025-12-10 — removed the global env deletion (#164).
- HEAD: the `PiAnthropic` subclass overrides `_shouldResolveDefaultCredentials()` → false, so the SDK never runs its own credential chain (`packages/ai/src/api/anthropic-messages.ts:329-339`; test `packages/ai/test/anthropic-federation-sdk.test.ts:107`).

**Lesson** — Never mutate process-global state for per-request auth. Pass credentials explicitly and disable SDK ambient discovery.

Related: [[credential-resolution]] · [[pi--credential-resolution|pi]] · [[env-credential-discovery-misfires]]
