---
type: failure
concepts: [subscription-oauth-auth, mcp-integration]
harnesses: [pi]
---
**Symptom** — Users were silently logged out when several pi instances ran at once:
- A stalled OAuth refresh held the credential lock indefinitely.
- After a user cancelled during a refresh, the next request failed with `refresh_token_invalidated` (Sign in with ChatGPT rotates refresh tokens).

**Root cause**
- Refresh tokens rotate. Concurrent refreshers race, and the losers overwrite or discard the only valid token.
- A refresh with no timeout blocks every waiter.
- `acbdc0d25` passed the caller's AbortSignal into `oauth.refresh`, so a cancellation after the provider had already rotated the token discarded the new token.

**Fix · [[pi]]**
- `6a8609ac5` 2026-01-05 — file lock plus re-read and expiry re-check under the lock (#466).
- `acbdc0d25` 2026-08-04 — refresh bounded by `DEFAULT_OAUTH_REFRESH_TIMEOUT_MS = 15_000` via `AbortSignal.any([signal, timeout])` (#7508).
- `bde882c74` 2026-10-04 — the caller's signal cancels only the **lock wait**. Refresh and persist run under the timeout only, "the provider may already have rotated the refresh token … a cancelled caller could discard the only valid refresh token" (`packages/ai/src/auth/resolve.ts:103-155`).
- HEAD `refreshStoredOAuthCredential` uses double-checked locking: an optimistic check, then `credentials.modify` re-checks `needsRefresh` under the lock. The same rule applies during model refresh (`packages/ai/src/models.ts:624-632`).

**Fix · [[pi]] (MCP OAuth, same behavior)** — `1d74741e1` 2026-09-29 "serialize MCP OAuth refreshes across processes": rotating refresh tokens (Cloudflare) were lost when two pi processes refreshed one MCP server at once, leaving the server needing sign-in. Fix: per-server `proper-lockfile` lock held from reading tokens to saving new ones, reuse of tokens another process already refreshed, stale-lock takeover when a holder is killed, and connection close waits for a running refresh (`packages/coding-agent/src/extensions/mcp/oauth.ts:34,46-53,169-173`; constants `REFRESH_REQUEST_TIMEOUT_MS = 15_000`, `REFRESH_LOCK_STALE_MS = 20_000`, `REFRESH_LOCK_WAIT_MS = 25_000`, `REFRESH_SKEW_MS = 30_000`). See [[pi--mcp-integration|pi MCP]].

**Fix · [[pi]] (CI, same behavior)** — `abe9c9d9f` 2026-07-06 issue-analysis workflow runs pi with a stored `PI_AUTH_JSON` and writes the refreshed `auth.json` back to the environment secret, refusing files without an `openai-codex` refresh token (`.github/workflows/issue-analysis.yml:418-452`). Header warns the login must be dedicated: Codex rotates refresh tokens on every refresh, so an auth.json shared with a developer machine "invalidates whichever copy refreshes second" → "OAuth refresh failed for openai-codex" (`issue-analysis.yml:34-37`). Cross-machine copies of a rotating credential cannot be fixed by a file lock.

**Lesson** — Once a non-idempotent remote side effect starts, finish and persist it regardless of caller cancellation, bounded by a timeout. Rotation demands cross-process mutual exclusion plus re-validation after acquiring the lock.

Related: [[subscription-oauth-auth]] · [[mcp-integration]] · [[pi--subscription-oauth-auth|pi]] · [[credential-file-lock-contention]] · [[abort-propagation]]
