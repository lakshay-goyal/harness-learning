---
type: failure
concepts: [subscription-oauth-auth]
harnesses: [pi]
---
**Symptom** — Copilot login tripped GitHub rate limits, and a 429 on `GET /models` aborted the login (#7850, #8121, #6187).

**Root cause** — Login enabled model policies with up to 30 POSTs, batched ×4 concurrently.

**Fix · [[pi]]**
- `d5278eaac` 2026-08-15 — enable policies sequentially.
- `55b0db4d3` 2026-08-19 — enable only `unconfigured` known tool-capable models; 429 retry max 2 within a 5 s budget; 5 s per-request timeout (#8254) (`packages/ai/src/auth/oauth/github-copilot.ts:135-166,373-432,467-480`).
- Token refresh re-fetches models with no retries (`:353-367`).

**Lesson** — Login-time side effects must be minimal, sequential and rate-aware.

Related: [[subscription-oauth-auth]] · [[pi--subscription-oauth-auth|pi]] · [[model-catalog]]
