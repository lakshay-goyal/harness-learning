---
type: failure
concepts: [auto-retry-backoff, http-transport-hardening]
harnesses: [codex]
---
**Symptom** — Rate-limit retries used local backoff instead of the server's delay (issue #2131); Azure's "Try again in 35 seconds" not parsed (#4161); later `Retry-After` HTTP headers and nested `response.error.headers` in streamed failures ignored; relative delays didn't subtract time spent propagating the error; WebSocket→HTTP fallback fired before the advised deadline; token-refresh retries stormed.

**Root cause** — Server advice parsed per transport and per message format (regex over prose), as a relative delay, and bypassed by the transport switch.

**Fix · [[codex]]**
- `41eb59a07d` 2025-08-13 delay parsed from message text; `d9118c04bf` 2025-11-02 fuzzy "try again in" regex.
- `d047c33a1b` 2026-06-26 avoid server token refresh retry storms.
- `9d8de19674` 2026-09-23 "Honor Retry-After and preserve server retry deadlines (#47641)" — parsed into a monotonic deadline at header receipt; expired advice = zero delay, never local backoff (`codex-rs/protocol/src/error.rs:449-458`; `codex-rs/http-client/src/retry_after.rs`).
- `6ba4bf9e64` 2026-09-30 "Honor server retry advice across Responses retries and fallback (#49441)" — overload/RetryLimit retry only with advice; quota/usage/policy stay terminal even with advice; WebSocket-upgrade rejection's Retry-After preserved before HTTP fallback (`codex-rs/core/src/responses_retry.rs:120-166`).
- `44dd77b71e` 2026-10-02 Retry-After in failed Responses events (nested headers preferred over message text); `f6cf05af1d` 2026-10-06 Retry-After in WebSocket error events.
- Advice never extends the retry budget (`codex-rs/core/src/responses_retry.rs:1-3`).

**Lesson** — Treat server advice as an absolute deadline captured at receipt, prefer structured headers over prose, carry it through every transport (SSE events, WebSocket errors, fallback), and never let it extend the budget.

Related: [[auto-retry-backoff]] · [[http-transport-hardening]] · [[retry-backoff-hygiene]] · [[retry-classifier-regex-sprawl]] · [[codex--auto-retry-backoff|codex]]
