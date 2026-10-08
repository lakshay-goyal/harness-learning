---
type: failure
concepts: [deferred-responses, session-affinity-cache-routing]
harnesses: [codex]
---
**Symptom** — Incremental WebSocket requests reference `previous_response_id`. When the server no longer held that response, the error `previous_response_not_found` failed the turn instead of resending the full request (`64dc1c7a01` 2026-07-22).

**Root cause** — Server-side continuation state is a cache scoped to one connection, with its own lifetime and eviction. The delta request had no stateless fallback.

**Fix · [[codex]]** — `64dc1c7a01` 2026-07-22 (#34763), "Retry websocket requests when the previous response is missing":
- `PREVIOUS_RESPONSE_NOT_FOUND_CODE` is mapped to a retryable error, "Previous response was not found. Retrying the full request." (`codex-rs/codex-api/src/endpoint/responses_websocket.rs:166-168,632-647`).
- The retry sends a full `response.create`. Full-request reset reasons are recorded at `codex-rs/core/src/client.rs:1979-1994`; which reason this path records is unverified.
- pi's Codex adapter got the same fix the same day (`c5dcb2600` 2026-07-22; [[persistent-connection-lifetime-exceeded]]).

**Lesson** — Any server-side-state shortcut, such as append-only or delta requests, needs a stateless full-resend fallback that runs automatically.

Related: [[session-affinity-cache-routing]] · [[deferred-responses]] · [[transport-fallback-after-partial-output]] · [[persistent-connection-lifetime-exceeded]] · [[codex--session-affinity-cache-routing|codex]] · [[incremental-request-diverges-from-history]]
