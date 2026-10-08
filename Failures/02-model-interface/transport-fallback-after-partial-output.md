---
type: failure
concepts: [http-transport-hardening, auto-retry-backoff]
harnesses: [pi, opencode, codex]
---
**Symptom** — Codex WebSocket failures surfaced as hard errors. Falling back blindly after output had started would duplicate or splice half-streamed output, and repeated WebSocket failures were retried on every request.

**Root cause** — There was no fallback policy distinguishing "nothing streamed yet" from "partial output emitted", and no memory of a broken transport.

**Fix · [[pi]]** — `370fdae6f` 2026-05-03 (#4133):
- Fall back to SSE only if no events were emitted; otherwise throw (`packages/ai/src/api/openai-codex-responses.ts:356-371`).
- Mark the session sticky-SSE (`:366,978-986`).
- Append a `provider_transport_failure` diagnostic `{configuredTransport, fallbackTransport, eventsEmitted, phase, requestBytes}`.
- `start` is emitted lazily on the first WebSocket event, so the fallback is invisible (`:1479-1491`).
- Aborts and non-transport errors (`CodexApiError`, `CodexProtocolError`) never fall back (`:353-355`).

**Fix · [[codex]]**
- Symptom: some proxies don't support WebSockets, and others answer `426 Upgrade Required`.
- `3b1cddf001` 2026-01-29: fall back to HTTP when WebSockets fail.
- `6d08298f4e` 2026-02-07: fall back on `UPGRADE_REQUIRED`.
- The fallback is sticky for the session (`codex-rs/core/src/client.rs:1030-1038,2296-2310`).
- It is the *last retry*, after stream retries are exhausted. It resets the retry counter but still waits for the server's Retry-After deadline (`6ba4bf9e64` 2026-09-30; `codex-rs/core/src/responses_retry.rs:120-139`).
- The full request is replayed over HTTPS. No explicit "events already emitted" guard appears in the findings (unverified).
**Fix · [[opencode]]** `14e0b9b17f` 2026-05-28: the Responses WebSocket transport fell back to HTTP only before the first event and failed hard after it. Now WS failures are retried up to `streamRetries ?? 5`, then the session is pinned to HTTP; a failure after the first event surfaces as a retryable error instead of replaying partial output (`packages/opencode/src/plugin/openai/ws-pool.ts:37,167`).

**Lesson** — Fall back only before the first event, or replay the whole request. Make the fallback sticky and observable, and never let a transport switch bypass server retry advice.
Related: [[http-transport-hardening]] · [[errors-as-stream-events]] · [[pi--http-transport-hardening|pi]] · [[persistent-connection-lifetime-exceeded]] · [[opencode--http-transport-hardening|opencode]]

Related: [[http-transport-hardening]] · [[errors-as-stream-events]] · [[pi--http-transport-hardening|pi]] · [[persistent-connection-lifetime-exceeded]] · [[codex--http-transport-hardening|codex]] · [[server-retry-advice-ignored]]
