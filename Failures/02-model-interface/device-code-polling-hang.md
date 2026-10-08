---
type: failure
concepts: [subscription-oauth-auth]
harnesses: [pi]
---
**Symptom** — Copilot device login appeared to hang after the user authorized in the browser, especially under WSL or VM clock drift.

**Root cause** — The client tracked `slow_down` by adding 5 s on its own side, which fell behind the server's ratchet. Polling started too early.

**Fix · [[pi]]**
- `e2ccdc850` 2026-07-01 — `waitBeforeFirstPoll` for Copilot (`packages/ai/src/auth/oauth/github-copilot.ts:263-311`).
- `8133c94db` 2026-07-03 — on `slow_down`, use the server-provided `interval` if present, else +5 s (RFC 8628 §3.5). Minimum 1 s, default 5 s. The timeout message mentions clock drift when `slow_down` was seen (`packages/ai/src/auth/oauth/device-code.ts:3-9,76-87,97`).

**Lesson** — In polling protocols, trust server-provided pacing over client bookkeeping.

Related: [[subscription-oauth-auth]] · [[pi--subscription-oauth-auth|pi]] · [[oauth-loopback-callback-failures]]
