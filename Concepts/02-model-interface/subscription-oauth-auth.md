---
type: concept
stage: model-interface
tier: candidate
aliases: [/login, OAuthAuth, lazyOAuth, refreshStoredOAuthCredential, isSubscription, subscription-oauth-login, loopback-oauth-with-paste-fallback, device-code-oauth, locked-oauth-refresh, non-cancellable-token-rotation, minted-short-lived-api-key, workload-identity-federation, credential-file-locking]
harnesses: [pi]
---
Log in with a consumer subscription or an org identity and use the resulting token as the model API credential. Login uses a PKCE loopback callback with a paste fallback, or a device code. Refresh is proactive, locked across processes, and double-checked. Rotated refresh tokens are always persisted.

## Why
- Users want to use subscriptions they already pay for (Claude Pro/Max, ChatGPT, Copilot, Grok, Kimi, Meta), not just API keys.
- Refresh tokens rotate:
  - Concurrent processes refreshing at once log each other out.
  - Cancelling a refresh after the provider has rotated discards the only valid token.

  See [[oauth-refresh-token-rotation-lost]].
- Short-lived tokens expire during long tool phases ([[credential-expires-mid-run]]).
- Sharing a credential file needs locking that never turns contention into "no credentials" ([[credential-file-lock-contention]]).
- Loopback login breaks in several environments: reserved ports, a fixed port already in use, a headless SSH session, a redirect_uri mismatch ([[oauth-loopback-callback-failures]]).
- Device-code polling falls behind under clock drift ([[device-code-polling-hang]]).
- Side effects at login trip rate limits ([[login-side-effects-rate-limited]]).
- Refreshing during availability checks slows the UI or blocks startup ([[credential-refresh-on-availability-path]]).

## Design space
- **Flow**
  - Loopback PKCE on a fixed port, falling back to a free port and then paste-only.
  - Device code (RFC 8628), with the server's `interval` honored on `slow_down`.
  - A permanent key exchange (OpenRouter).
  - An identity token exchanged for a short-lived key that is minted again on each refresh (Meta).
  - Workload identity federation.
- **Refresh timing**
  - On a 401.
  - Proactively, when less than 5 min of validity remains. *pi chose this.*
- **Concurrency**
  - None.
  - A file lock with re-read and re-check under the lock (double-checked locking). *pi chose this:* 6a8609ac5.
- **Cancellation**
  - Refresh honors the caller's abort. *pi tried this:* acbdc0d25.
  - The caller can cancel only the wait for the lock. Once a refresh starts it runs to completion under a timeout and is persisted. *pi chose this:* bde882c74.
- **Failure handling**
  - Throw at startup.
  - Keep the stored credential, report "not configured" and let the user re-login. *pi chose this:* d2f3b42de.
- **Credential read path**
  - Lock on every read.
  - A stat-based revision fast path plus single-flight reloads. *pi chose this.*
- **Identity**
  - Plain OAuth.
  - Impersonate the vendor's first-party client (see [[provider-identity-shim]]).

## Implementations
- [[pi--subscription-oauth-auth|pi]] — pi-ai `OAuthAuth` and `lazyOAuth`, with flows for Anthropic, Codex, ChatGPT, Copilot, OpenRouter, xAI, Kimi, Meta and Radius. `resolve.ts` does the locked refresh. Coding-agent `AuthStorage` is `auth.json` with proper-lockfile.

## Failures
- [[oauth-refresh-token-rotation-lost]]
- [[credential-expires-mid-run]]
- [[credential-file-lock-contention]]
- [[oauth-loopback-callback-failures]]
- [[device-code-polling-hang]]
- [[login-side-effects-rate-limited]]
- [[credential-refresh-on-availability-path]]

## Related
[[credential-resolution]] · [[provider-identity-shim]] · [[custom-provider-registration]] · [[model-catalog]] · [[mcp-integration]]
