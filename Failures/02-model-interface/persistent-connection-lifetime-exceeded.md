---
type: failure
concepts: [http-transport-hardening]
harnesses: [pi]
---
**Symptom** — Long Codex sessions failed on the cached WebSocket:
- connection-limit failures (#6268);
- `websocket_connection_limit_reached` (#5973);
- `previous_response_not_found` after continuation state was lost (#6931).

**Root cause** — The backend enforces a hard 60-minute WebSocket lifetime and server-side capacity limits, and connection-scoped `previous_response_id` state disappears with the connection.

**Fix · [[pi]]**
- `d0e0b84cb` 2026-06-23 — `websocket_connection_limit_reached` before start → retry once (`packages/ai/src/api/openai-codex-responses.ts:349-352`).
- `23d146261` 2026-07-03 — rotate at `SESSION_WEBSOCKET_MAX_AGE_MS = 55 min`; idle TTL 5 min (`:866-867,1052-1073`).
- `c5dcb2600` 2026-07-22 — `previous_response_not_found` → retry once with the continuation already cleared, so full context is sent (`:344-348`).
- Open: does this retry `continue` even after events were emitted? (`:344-348`, unverified)

**Lesson** — Pool reuse needs a max age below the server's hard limit. Stateful-continuation transports need a one-shot stateless retry.

Related: [[http-transport-hardening]] · [[pi--http-transport-hardening|pi]] · [[connection-cache-shared-across-accounts]] · [[transport-fallback-after-partial-output]]
