---
type: failure
concepts: [http-transport-hardening]
harnesses: [pi]
---
**Symptom** — Codex WebSocket failures surfaced as hard errors. Falling back blindly after output had started would duplicate or splice half-streamed output, and repeated WebSocket failures were retried on every request.

**Root cause** — There was no fallback policy distinguishing "nothing streamed yet" from "partial output emitted", and no memory of a broken transport.

**Fix · [[pi]]** — `370fdae6f` 2026-05-03 (#4133):
- Fall back to SSE only if no events were emitted; otherwise throw (`packages/ai/src/api/openai-codex-responses.ts:356-371`).
- Mark the session sticky-SSE (`:366,978-986`).
- Append a `provider_transport_failure` diagnostic `{configuredTransport, fallbackTransport, eventsEmitted, phase, requestBytes}`.
- `start` is emitted lazily on the first WebSocket event, so the fallback is invisible (`:1479-1491`).
- Aborts and non-transport errors (`CodexApiError`, `CodexProtocolError`) never fall back (`:353-355`).

**Lesson** — Fall back only before the first event, make the fallback sticky, and make it observable.

Related: [[http-transport-hardening]] · [[errors-as-stream-events]] · [[pi--http-transport-hardening|pi]] · [[persistent-connection-lifetime-exceeded]]
