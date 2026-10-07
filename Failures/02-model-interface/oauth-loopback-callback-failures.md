---
type: failure
concepts: [subscription-oauth-auth]
harnesses: [pi]
---
**Symptom** — Browser and loopback OAuth logins failed in several ways:
- The manual-paste exchange for Anthropic failed, and refresh sent `scope`.
- On Windows the browser showed "localhost refused to connect" (port 53692 reserved by Hyper-V/WSL).
- With Sign in with ChatGPT, "OAuth state mismatch" occurred when another login, or the Codex CLI, held port 1455.
- Headless, SSH and container users couldn't complete the browser flow at all.

**Root cause**
- The token exchange used a different `redirect_uri` than the authorize request.
- Fixed loopback ports can be reserved or taken.
- Browsers keep spare pre-opened connections that deliver the NEXT login's callback to the old server.
- There was no paste or device fallback.

**Fix · [[pi]]**
- `c7309aeda` 2026-03-14 — fetch instead of curl for the token exchange.
- `e645995a4` 2026-03-15 — the manual exchange keeps the same localhost `redirect_uri`, and refresh omits `scope` (#2169) (`packages/ai/test/anthropic-oauth.test.ts:43,145`).
- `ebc60aa82` 2026-04-20 — `PI_OAUTH_CALLBACK_HOST` bind override (#3396/#3409).
- `9d5fb70b7` 2026-05-28 — Codex device-code login (#4911) (`packages/ai/src/auth/oauth/openai-codex.ts:413-431`).
- `61da9e2f3` 2026-07-27 — OpenRouter manual redirect URL fallback (#7114).
- `7a11fe1c7` 2026-09-30 — Anthropic "Copy code login (headless)" (#10194) (`packages/ai/src/auth/oauth/anthropic.ts:278-300`).
- `eeac84ca9` 2026-10-02 — Sign in with ChatGPT hard-fails on EADDRINUSE ("probably by an unfinished login in another pi session or by the Codex CLI") and calls `closeAllConnections()` after login (`packages/ai/src/auth/oauth/openai-chatgpt.ts:241-248,288-297`).
- `8d8ae2fc2` 2026-10-07 — free-port fallback `startCallbackServer(0)`, then paste-only (#10571) (`packages/ai/src/auth/oauth/anthropic.ts:142-157`).
- HEAD `waitForCallbackOrManualInput` races the callback against a pasted code (`packages/ai/src/auth/oauth/callback-server.ts:155-183`).

**Lesson** — Loopback OAuth needs an exact `redirect_uri` match, a preferred port with fallback, explicit port-conflict errors, connection cleanup, and a concurrent paste or device-code path.

Related: [[subscription-oauth-auth]] · [[pi--subscription-oauth-auth|pi]] · [[device-code-polling-hang]]
