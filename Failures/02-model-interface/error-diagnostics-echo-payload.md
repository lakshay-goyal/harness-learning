---
type: failure
concepts: [errors-as-stream-events, secret-handling]
harnesses: [codex]
---
**Symptom** — Error diagnostics leaked payloads:
- Model-catalog decode errors included the full response body and payload values.
- Connection errors exposed request URLs.

This is the inverse of [[provider-error-body-hidden]].

**Root cause** — Default error formatting from JSON and HTTP libraries embeds the input that failed and the target URL. Those strings then flow into logs, telemetry and the UI.

**Fix · [[codex]]**
- `977193486d` 2026-09-16: report only the JSON error category, line, column and byte count. A catalog fetch deadline is classified as `RequestTimeout`.
- `5a0d0929e2` 2026-08-07: connection errors are classified as `ConnectionFailed` without leaking URLs.
- SSE parse errors are logged with the same bounded fields and the event is skipped, not treated as fatal (`codex-rs/codex-api/src/sse/responses.rs:573-585`).

**Lesson** — Error diagnostics should be structural (category, position, size, status, request id), not echoes of the payload. Payloads may hold secrets or user data, and they bloat logs.

Related: [[errors-as-stream-events]] · [[secret-handling]] · [[provider-error-body-hidden]] · [[codex--errors-as-stream-events|codex]]
