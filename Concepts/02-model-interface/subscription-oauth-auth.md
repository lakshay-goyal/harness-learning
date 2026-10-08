---
type: concept
stage: model-interface
tier: must-have
aliases: [/login, OAuthAuth, lazyOAuth, refreshStoredOAuthCredential, isSubscription, subscription-oauth-login, loopback-oauth-with-paste-fallback, device-code-oauth, locked-oauth-refresh, non-cancellable-token-rotation, minted-short-lived-api-key, workload-identity-federation, credential-file-locking, OAUTH_POLLING_SAFETY_MARGIN_MS, CodexAuthPlugin, opencode-anthropic-auth, codex login, AuthManager, CodexAuth, UnauthorizedRecovery, cli_auth_credentials_store, Sign in with ChatGPT]
harnesses: [pi, opencode, codex]
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
- **First-party variant** (codex)
  - Loopback on fixed port 1455 with a *second registered* port, 1457. ✔ codex
  - Device code with a 15 min cap. ✔ codex
  - Proactive refresh 5 min before JWT `exp` (8 days when no `exp`), plus a reactive 401 state machine: reload `auth.json` (account-id guarded) → refresh → give up. ✔ codex
  - Concurrency: reload from shared storage before refreshing, with no file lock in the findings. ✔ codex
  - Permanent refresh failures (`refresh_token_reused`, `invalid_grant`) are cached with specific relogin messages. ✔ codex
  - Storage: file (0600), OS keyring, auto or ephemeral host-supplied tokens. ✔ codex
  - Identity is first-party, so no shim is needed ([[provider-identity-shim]]). ✔ codex
- **Vendor scope**: ChatGPT, Copilot, GitLab, xAI only; Claude subscription removed after a legal request (opencode 1.3.0, `1ac1a0287c`).

## Implementations
- [[pi--subscription-oauth-auth|pi]] — pi-ai `OAuthAuth` and `lazyOAuth`, with flows for Anthropic, Codex, ChatGPT, Copilot, OpenRouter, xAI, Kimi, Meta and Radius. `resolve.ts` does the locked refresh. Coding-agent `AuthStorage` is `auth.json` with proper-lockfile.
- [[codex--subscription-oauth-auth|codex]] — `codex login` PKCE (1455→1457) or device code; `AuthManager` proactive and 401 recovery; keyring/file/ephemeral storage; agent, workload and gateway identities.
- [[opencode--subscription-oauth-auth|opencode]] — built-in Codex (port 1455), Copilot, GitLab, xAI device code; no Claude Pro/Max since 2026-03.

## Failures
- [[vendor-prompt-copy-legal-exposure]]
- [[oauth-refresh-token-rotation-lost]]
- [[credential-expires-mid-run]]
- [[credential-file-lock-contention]]
- [[oauth-loopback-callback-failures]]
- [[device-code-polling-hang]]
- [[login-side-effects-rate-limited]]
- [[credential-refresh-on-availability-path]]
- [[api-key-overrides-subscription-auth]] (02-model-interface) — Users logged in with a subscription (OAuth) were billed pay-as-you-go because an API key in settings.json…
- [[oauth-issuer-mixup-accepted]] (07-safety) — pi's MCP OAuth client exchanged an authorization code from a response naming a different issuer than the…

## Related
[[credential-resolution]] · [[provider-identity-shim]] · [[custom-provider-registration]] · [[model-catalog]] · [[mcp-integration]] · [[subscription-usage-limits]]
