---
type: implementation
harness: codex
concept: subscription-oauth-auth
commit: 622e9e3696
files: [codex-rs/protocol/src/auth.rs:9-38, codex-rs/login/src/server.rs:78-80, codex-rs/login/src/server.rs:194, codex-rs/login/src/device_code_auth.rs:68, codex-rs/login/src/device_code_auth.rs:107-108, codex-rs/login/src/auth/manager.rs:112, codex-rs/login/src/auth/manager.rs:203-212, codex-rs/login/src/auth/manager.rs:1718, codex-rs/login/src/auth/manager.rs:1850-1862, codex-rs/login/src/auth/manager.rs:3001-3023, codex-rs/login/src/auth/storage.rs:225, codex-rs/login/src/auth/storage.rs:243-258, codex-rs/config/src/types.rs:136-146, codex-rs/core/src/client.rs:2640-2700, codex-rs/model-provider-info/src/lib.rs:426-440]
---
[[subscription-oauth-auth]] in [[codex]]. Codex is the vendor's own client, so "Sign in with ChatGPT" is first-party and needs no identity shim ([[provider-identity-shim]]).

## Mechanism
- **Auth modes** (`codex-rs/protocol/src/auth.rs:9-38`):
  - `ApiKey`;
  - `Chatgpt`: Codex-managed OAuth, persisted and refreshed;
  - `ChatgptAuthTokens`: supplied by the host app;
  - `Headers`, `AgentIdentity`, `PersonalAccessToken`, `BedrockApiKey`, `BedrockAccessKeys`.

  ChatGPT-family modes route to `https://chatgpt.com/backend-api/codex` instead of `https://api.openai.com/v1` (`codex-rs/model-provider-info/src/lib.rs:426-440`).
- **Login: PKCE loopback.**
  - The server listens on 127.0.0.1 port 1455 and falls back to registered port 1457 when 1455 is busy (`codex-rs/login/src/server.rs:78-80,194`). Cursor and Codex Desktop both use 1455 (`8d5da3ffe5`).
  - OAuth client id is `CLIENT_ID` = `app_EMoamEEZ73f0CkXaXp7hrann` (`codex-rs/login/src/auth/manager.rs:1718`).
  - Refresh endpoint is `https://auth.openai.com/oauth/token` (`codex-rs/login/src/auth/manager.rs:212`).
- **Login: device code.** `/deviceauth/usercode`, then poll `/deviceauth/token` at the server's interval, waiting at most 15 min (`codex-rs/login/src/device_code_auth.rs:68,107-108`).
- **Revocation.** Existing tokens are revoked before a new login and on logout (`22f7ef1cb7` 2026-04-16, `d1aaf789ad` 2026-06-12).
- **Proactive refresh** (`codex-rs/login/src/auth/manager.rs:203-204,3001-3023`):
  - refresh if the access-token JWT `exp` is within `CHATGPT_ACCESS_TOKEN_REFRESH_WINDOW_MINUTES` = 5;
  - with no `exp`, refresh if `last_refresh` is older than `TOKEN_REFRESH_INTERVAL` = 8 days.
- **Reactive 401 recovery** is a state machine (`codex-rs/login/src/auth/manager.rs:1850-1862`):
  - Managed ChatGPT auth: (1) reload `auth.json` from disk, only if the account id matches; (2) OAuth refresh; (3) give up.
  - API-key auth: no recovery.
  - External auth: ask the host to refresh, once.
  - Provider-level recovery and the UI events `AuthRecoveryStarted/Completed` are in `codex-rs/core/src/client.rs:2640-2700`.
- **Permanent refresh failures.** Expired, reused, revoked, account mismatch and `invalid_grant` each get a specific relogin message (`codex-rs/login/src/auth/manager.rs:206-211`). They are cached, so no further refresh calls are made.
- **Storage.** `cli_auth_credentials_store` (`codex-rs/config/src/types.rs:136-146`; `codex-rs/login/src/auth/storage.rs:225,243-258`) is one of:
  - `file`: the default, `$CODEX_HOME/auth.json`, mode 0600;
  - `keyring`: service "Codex Auth", key `cli|<sha256(codex_home)[..16]>`;
  - `auto`;
  - `ephemeral`.
- **Workspace restrictions.** `forced_chatgpt_workspace_id` and allowed login methods are enforced when auth loads.
- **Other identity paths:**
  - `codex-rs/agent-identity`: programmatic agent JWT, with a 1 h bootstrap-failure cooldown (`codex-rs/login/src/auth/manager.rs:112`).
  - `codex-rs/workload-identity`: OIDC workload token exchange, prod token URL `https://auth.openai.com/oauth/token` (`codex-rs/login/src/auth/workload_identity.rs:32-33`).
  - Gateway OAuth for custom providers: refresh skew 30 s, HTTP timeout 20 s, browser timeout 180 s (`codex-rs/login/src/gateway_auth.rs:54-55`; `codex-rs/login/src/gateway_auth_callback.rs:15`).

## Constants
| name | value | path:line |
|---|---|---|
| `CHATGPT_ACCESS_TOKEN_REFRESH_WINDOW_MINUTES` | 5 | `codex-rs/login/src/auth/manager.rs:204` |
| `TOKEN_REFRESH_INTERVAL` (no `exp`) | 8 days | `codex-rs/login/src/auth/manager.rs:203` |
| login callback ports | 1455, fallback 1457 | `codex-rs/login/src/server.rs:78-80` |
| device-code max wait | 15 min | `codex-rs/login/src/device_code_auth.rs:108` |
| gateway OAuth refresh skew | 30 s | `codex-rs/login/src/gateway_auth.rs:54` |
| agent identity bootstrap failure cooldown | 1 h | `codex-rs/login/src/auth/manager.rs:112` |

## Evolution
- `d63e44ae29` 2025-08-25: the retry loop kept using the old token after a refresh; fixed.
- `999576f7b8` 2026-02-18, `f55f5c258f` 2026-03-23: reload before refreshing, since another process may have rotated tokens on disk.
- `7dc2cd2ebe` 2026-03-24: proactive refresh based on JWT `exp`.
- `88694e8417` 2026-03-24: refresh-storm fix.
- `2c67a27a71` 2026-03-25: duplicate refresh in `getAuthStatus` removed.
- `22f7ef1cb7` 2026-04-16: revoke on logout.
- `8d5da3ffe5` 2026-04-29: fallback port 1457.
- `6111791d0b` 2026-05-27: `refresh_token_reused` (400) cached as permanent.
- `e5afe5bf8c` 2026-05-28: 5-minute refresh window.
- `d1aaf789ad` 2026-06-12: revoke before a new login.
- `fdc23b93b8` 2026-08-19: `invalid_grant` cached as permanent.

## Versus pi
- [[pi--subscription-oauth-auth|pi]] uses a file lock plus double-checked refresh under the lock, and refresh is non-cancellable once started.
- Codex has no cross-process lock in the findings. It relies on reloading `auth.json` from disk with an account-id guard before refreshing, and on caching permanent failures.
- Both use a 5-minute proactive window.

## Failures
- [[oauth-refresh-token-rotation-lost]]
- [[oauth-loopback-callback-failures]]
