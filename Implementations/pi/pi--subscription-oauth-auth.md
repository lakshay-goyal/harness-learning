---
type: implementation
harness: pi
concept: subscription-oauth-auth
commit: b30a6dd77
files: [packages/ai/src/auth/types.ts:216, packages/ai/src/auth/resolve.ts:102, packages/ai/src/auth/helpers.ts:33, packages/ai/src/auth/oauth/callback-server.ts:50, packages/ai/src/auth/oauth/device-code.ts:5, packages/ai/src/auth/oauth/pkce.ts:21, packages/ai/src/auth/oauth/anthropic.ts:13, packages/ai/src/auth/oauth/openai-codex.ts:22, packages/ai/src/auth/oauth/openai-chatgpt.ts:15, packages/ai/src/auth/oauth/github-copilot.ts:10, packages/ai/src/auth/oauth/meta.ts:19, packages/ai/src/auth/oauth/radius.ts:18, packages/coding-agent/src/core/auth-storage.ts:52]
---
[[subscription-oauth-auth]] in [[pi]].

## Mechanism

### Contract
- `OAuthAuth{name, isSubscription?, loginLabel?, login, refresh(cred, signal), toAuth(cred)}` — split so `Models` owns locked refresh; `toAuth` may return per-credential `baseUrl` (Copilot) or `headers` (Kimi) (`packages/ai/src/auth/types.ts:216-245`). `OAuthCredential{type:"oauth", refresh, access, expires, ...extra}` = "the shape of today's auth.json" (`types.ts:17-37`).
- Login UX: `AuthInteraction{prompt(text|secret|select|manual_code), notify(info|auth_url|device_code|progress)}`; per-prompt `signal` lets a `manual_code` prompt race a local callback server and be cancelled when the callback wins (`types.ts:119-161`). `LoginOptions.getDeviceId` (OpenAI agent host id), `agentName` (OpenAI agent hint / Codex originator) (`:201-214`).
- `lazyOAuth({name, load})` defers Node-only flow code via bundler-opaque dynamic import (`auth/helpers.ts:33-59`); OAuth flow modules loaded via variable import specifier + `.ts→.js` rewrite; Bun binaries register static loaders (`auth/oauth/load.ts:1-80`; `6442536b1`).
- `isSubscription` drives `isUsingSubscription` (`packages/coding-agent/src/core/model-runtime.ts:540-542`; `b0bd0ff9d`).

### Locked refresh (pi-ai `refreshStoredOAuthCredential`)
- Proactive refresh when < `DEFAULT_OAUTH_MINIMUM_VALIDITY_MS = 5 min` left (`auth/resolve.ts:102, 161-191`).
- **Double-checked locking**: optimistic expiry check, then `credentials.modify` re-checks `needsRefresh` under lock → concurrent requests/processes refresh once (`resolve.ts:104-155`). `CredentialStore.modify` = only write path, serialized per provider, cross-process with file lock (`auth/types.ts:50-94`); `InMemoryCredentialStore` per-provider promise chain (`auth/credential-store.ts:9-67`).
- **Non-cancellable once started**: lock wait cancellable; refresh+persist ignore caller signal, bounded by `DEFAULT_OAUTH_REFRESH_TIMEOUT_MS = 15_000`, because "the provider may already have rotated the refresh token ... a cancelled caller could discard the only valid refresh token" (`resolve.ts:103, 110-114, 140`; `bde882c74`).
- Failure → `ModelsError("oauth")`, stored credential preserved for re-login (`packages/ai/src/models.ts:302-305`); no env fallback after failed refresh (`resolve.ts:27-32`). `minOAuthValidityMs` override for bearer export (`pi auth print`, 30 min) enforces post-refresh validity (`resolve.ts:177-183`; `packages/coding-agent/src/cli/credential-print.ts:7`; `99e34013d`).
- Model refresh: OAuth refresh "survives cancellation ... so a rotated refresh token is always persisted" (`packages/ai/src/models.ts:624-632`).
- Credentials resolved **per LLM call** (`getApiKey` per request, `1167e8445` #223 — Copilot ~30 min tokens expired in long tool phases).

### auth.json storage (coding-agent `AuthStorage`)
- `~/.pi/agent/auth.json` (`getAgentDir()`, `auth-storage.ts:52`; `PI_CODING_AGENT_DIR` override), `Record<providerId, Credential>` validated in `ReadOnlyAuthStorage.load` (`:228-252`). Mode `0o600` only on creation "so administrator-managed modes and ACLs remain intact" (`:24-25, 63-67`; `135fb545f`); dir `0o700` (`:59`).
- Sync lock: `lockSync` retried 10× with 20 ms busy-wait (`:69-94`; `58f8fcd8f` #1871). Async: `stale 30s`, own retry loop with 30 s deadline, backoff `min(10·2^n, 1000)·(1+rand)` capped by remaining (`:116-155`; `4d68d9355`); abort-aware sleep, release if aborted after acquire (`:149-152`). Compromised lock (`onCompromised`) checked before read, after fn, after write → throws instead of crashing (`:157-200`; `ddd5a65c7` #1322).
- Reads: `getFileRevision = dev:ino:size:mtimeNs:ctimeNs` (`packages/coding-agent/src/utils/paths.ts:36-43`); unchanged revision → no lock; else **single-flight reload** shared by readers, aborted when last reader leaves; process-wide state per path (`auth-storage.ts:39, 401-439`). Arc `57cde8690` → `d2be68dbe` → `2c79ce453` → `7cf90c1d1` (models store).
- `modify` = read-modify-write under lock; `undefined` = no write (`:449-471`); save failures surfaced to `/login` (`f8bec25f3` #6223). Credential mutations serialized per provider and followed by local resync (`model-runtime.ts:572-612`; [[pi--model-catalog|model-catalog]]).

### Shared flow building blocks
- PKCE S256 via WebCrypto (32 random bytes → base64url verifier) (`auth/oauth/pkce.ts:21-34`; `c10fc1e08` removed Node crypto).
- **Loopback callback server** (`auth/oauth/callback-server.ts:50-148`): GET + exact path else 404; state mismatch 400; second callback 409; provider `error`/`error_description` surfaced; `complete(code)` runs token exchange **before** rendering so failures show in browser (502); port 0 supported; IPv6 bracketed in redirectUri; `cache-control: no-store`; optional timeout. `waitForCallbackOrManualInput` races callback vs pasted code/URL (SSH/headless) (`:155-183`); pasted input parsed as URL, `code#state`, query string, or bare code (anthropic `:29-57`). Bind host `PI_OAUTH_CALLBACK_HOST` default 127.0.0.1 (`ebc60aa82` #3396). Shared server + sign-in page (`4df157433`).
- **Device-code poller** (`auth/oauth/device-code.ts`): min interval 1 s, default 5 s (RFC 8628 §3.2), `slow_down` → server `interval` if present else +5 s (§3.5) (`:5-9, 76-87`; `8133c94db` WSL/VM clock drift "login appears to hang"); timeout message mentions clock drift when slow_down seen (`:3-4, 97`). Verification URIs must be http(s) (anti-`open`-executable) (copilot `:242-252`, meta `:57-67`, xai `:49-62`).

### Per-provider flows
| provider (label) | grant | client id / endpoints | scopes / extras | storage & expiry | evidence |
|---|---|---|---|---|---|
| Anthropic (Claude Pro/Max), `isSubscription` | PKCE browser loopback (default) or "Copy code login (headless)" | CLIENT_ID base64-obfuscated, `atob` (`anthropic.ts:13-14`); authorize `https://claude.ai/oauth/authorize`, token `https://platform.claude.com/v1/oauth/token` (`:15-16`, moved from console.anthropic.com `92882dc4c`) | `org:create_api_key user:profile user:inference user:sessions:claude_code user:mcp_servers user:file_upload` (`:26-27`); **state = PKCE verifier** (`:149, 168, 210`); `code=true` | loopback `PI_OAUTH_CALLBACK_HOST||127.0.0.1`:53692 `/callback`, redirect host `localhost` → fallback free port (`startCallbackServer(0)`) → paste-only (`:17-22, 142-157`; `8d8ae2fc2` #10571); copy-code redirect `https://platform.claude.com/oauth/code/callback`, paste `code#state` (`:200-235`; `7a11fe1c7`); exchange JSON POST, 30 s timeout ⊕ caller signal (`:76-94`); `expires = now + expires_in·1000 − 5 min` (`:136, 274`); refresh omits `scope` (`e645995a4`); manual paste must reuse same localhost redirect_uri (`e645995a4`); fetch not curl (`c7309aeda`); `toAuth → {apiKey: access}` (`:304-306`) | `providers/anthropic.ts:74-90` |
| OpenAI Codex (ChatGPT Plus/Pro, legacy provider `openai-codex`) | PKCE browser or device code (`9d5fb70b7`) | `app_EMoamEEZ73f0CkXaXp7hrann`; `https://auth.openai.com/oauth/authorize`, `/oauth/token`; redirect `http://localhost:1455/auth/callback` (`openai-codex.ts:22-34`) | `openid profile email offline_access`; 16-byte hex state; `id_token_add_organizations=true`, `codex_cli_simplified_flow=true`, `originator=<agentName|pi>` (`:289-308`) | fixed port 1455 (shared with Codex CLI); bind failure → manual paste (state checked) (`:359-400`); device: `POST /api/accounts/deviceauth/usercode`, poll `/deviceauth/token` (403/404 = pending), 15-min timeout, exchange with `DEVICE_REDIRECT_URI` (`:187-287, 341-357`); must contain `chatgpt_account_id` claim (`:310-330`); `expires = now + expires_in` (no margin, `:141`); `toAuth → {apiKey: access}` (`:435-437`) | `providers/openai-codex.ts:7-22` |
| Sign in with ChatGPT (for `openai` provider) | PKCE + dynamic client registration | `client_id=dynamic_agent_client`, issued id returned in callback & stored (`openai-chatgpt.ts:15-16, 53-62`); `ext_agent_host_id=urn:uuid:<deviceId>` (`:225-231`); `resource=https://api.openai.com/v1` | `openid profile email offline_access resource.invoke chatgpt.tokens.use.direct`, nonce, `agent_name_hint` (`:19-27, 250-263`); grant must include `chatgpt.tokens.use.direct` + id_token (`:162-206`) | margin 3 min (`:29, 177`); refresh needs stored clientId else "reconnect ChatGPT" (`:208-223`); port 1455 EADDRINUSE → hard fail "probably by an unfinished login in another pi session or by the Codex CLI" (`:241-248`; `eeac84ca9`); `closeAllConnections()` after login so spare browser sockets don't deliver the next login's callback ("OAuth state mismatch") (`:288-297`); token sent direct to api.openai.com Responses with fields stripped (`openai-responses.ts:328-346`) | `02eed88fd` |
| GitHub Copilot | device code | client id base64 `Iv1.b507a08c87ecfe98` (VS Code Copilot app) (`github-copilot.ts:10-11`); impersonation headers — see [[pi--provider-identity-shim\|provider-identity-shim]] | prompt for GHE domain (blank = github.com) (`:434-445`); `POST https://<domain>/login/device/code` scope `read:user` (`:206-220`); poll `/login/oauth/access_token` with `waitBeforeFirstPoll` (`:263-311`; `e2ccdc850`) | GitHub token → Copilot token `GET https://api.<domain>/copilot_internal/v2/token`; `refresh`=GitHub token, `access`=Copilot token, `expires = expires_at·1000 − 5 min`, `enterpriseUrl` (`:313-348`); base URL from token `proxy-ep=proxy.X` → `https://api.X`, fallback `https://copilot-api.<enterprise>`, default `https://api.individual.githubcopilot.com` (`:64-87, 501-506`); model availability `GET {base}/models`, drop `tool_calls:false`, picker-enabled & policy≠disabled, Individual-only fallback (`:93-133, 174-177`); login enables `unconfigured` known models **sequentially** with 429 retry max 2 / 5 s budget, 5 s per-request timeout (`:135-166, 373-432, 467-480`; `d5278eaac`, `55b0db4d3`); refresh refetches models without retries (`:353-367`); stored `availableModelIds` filter (`providers/github-copilot.ts:19-27`) | |
| OpenRouter | PKCE, **no state**: unguessable callback path `/oauth/callback/<uuid>` on ephemeral port (`openrouter.ts:113-124`) | exchange `https://openrouter.ai/api/v1/auth/keys` → permanent user API key (`:58-111`) | manual paste fallback (`61da9e2f3`) | `expires: MAX_SAFE_INTEGER`, refresh no-op (`:159-169`); login 5 min, exchange 30 s (`:21-22`) | |
| xAI | device code `https://auth.x.ai/oauth2/device/code` | `referrer: pi` | `openid profile email offline_access grok-cli:access api:access` (`xai.ts:8-11, 145-159`); prefers `verification_uri_complete` (`:201-211`) | refresh may omit refresh_token (reuse previous) (`:128-143`); default lifetime 3600 s, skew 5 min (`:12-14`) | |
| Kimi Code | RFC 8628 device grant `https://auth.kimi.com` (override `KIMI_CODE_OAUTH_HOST`/`KIMI_OAUTH_HOST`) (`kimi-coding.ts:14-39`) | | 30 s request timeout (`:18, 41-43`); 15-min device timeout | refresh retries 3× exponential on 429/5xx; 401/403/`invalid_grant` → "unauthorized" (Models clears credential, prompts re-login) (`:210-265`); `toAuth → headers:{Authorization: Bearer}` because provider uses anthropic-messages (`:293-295`) | `providers/kimi-coding.ts:7-24` |
| Meta (Muse subscription), `isSubscription` | RFC 8628 at `auth.meta.com`, Muse Code CLI client id `1031625952748946` (`auth/oauth/meta.ts:19-23`) | | identity token **not usable for inference, not renewable** (refresh → 404) | **minted short-lived key**: exchange at `https://api.meta.ai/muse-code/key` → Model API key ~24 h; stored `{refresh: identityToken, access: apiKey, expires: now+24h}` so the generic refresher re-mints (`:24-25, 145-174, 203`); mint 401/403 → "Meta session expired… Run `/login meta`" (`:159-164`); missing key with `action_url` → setup URL (`:168-172`); 30 s per request (`:26, 36-38`); aborted login normalized to "Login cancelled" because UI matches that string (`:189-193`) | `providers/meta.ts:7-23`; `b73412a37` #9096 |
| Radius gateway | PKCE browser (discover `authorizationEndpoint` from `GET <gateway>/v1/oauth`) or device `POST /v1/oauth/device` | client `pi-gateway`, callback `http://127.0.0.1:1456/oauth/callback` (`radius.ts:18-27`) | `gateway offline_access`, random state, `handoff=url` (`:41-58, 134-187`); device maps `authorization_pending|slow_down|expired_token|access_denied` (`:189-268`) | `TOKEN_EXPIRY_SKEW_MS = 60_000` (`:22, 129`); refresh grant (`:304-315`); all endpoints on configured gateway (`a9f5b1c12`); legacy catalog cached in credential `gatewayConfig` (see [[pi--model-catalog\|model-catalog]]) | |
- Removed: Gemini CLI + Antigravity (Cloud Code Assist OAuth) providers (`fe66edd94`, −996 lines; reason unverified). Anthropic OAuth removed and restored same day (`f5e6bcac1` → `19b566334`, 2026-01-09; reason unverified).
- Durable/remote: Radius relay re-resolves credentials per connection attempt, refreshing if <5 min validity (`radius-auth.ts:16, 44-50`).

## Constants
| name | value | path:line |
|---|---|---|
| `DEFAULT_OAUTH_MINIMUM_VALIDITY_MS` | 5 min | packages/ai/src/auth/resolve.ts:102 |
| `DEFAULT_OAUTH_REFRESH_TIMEOUT_MS` | 15000 | packages/ai/src/auth/resolve.ts:103 |
| Bearer min expiry for `auth print` | 30 min | packages/coding-agent/src/cli/credential-print.ts:7 |
| Anthropic callback port | 53692 | packages/ai/src/auth/oauth/anthropic.ts:20 |
| OpenAI (Codex/ChatGPT) callback port | 1455 | packages/ai/src/auth/oauth/openai-codex.ts:26; openai-chatgpt.ts:23 |
| Radius callback port | 1456 | packages/ai/src/auth/oauth/radius.ts:19 |
| Anthropic expiry margin | 5 min | packages/ai/src/auth/oauth/anthropic.ts:136,274 |
| Copilot expiry margin | 5 min | packages/ai/src/auth/oauth/github-copilot.ts:345 |
| ChatGPT `EXPIRY_MARGIN_MS` | 3 min | packages/ai/src/auth/oauth/openai-chatgpt.ts:29 |
| xAI skew / default lifetime | 5 min / 3600 s | packages/ai/src/auth/oauth/xai.ts:13-14 |
| Radius `TOKEN_EXPIRY_SKEW_MS` | 60000 | packages/ai/src/auth/oauth/radius.ts:22 |
| Device poll min / default / slow_down | 1000 ms / 5 s / +5000 ms | packages/ai/src/auth/oauth/device-code.ts:5,7,9 |
| Device-code timeout (Codex, Kimi) | 15 min | packages/ai/src/auth/oauth/openai-codex.ts:31; kimi-coding.ts:16 |
| Kimi refresh retries / timeout | 3 / 30 s | packages/ai/src/auth/oauth/kimi-coding.ts:18-19 |
| Meta API key lifetime | 24 h | packages/ai/src/auth/oauth/meta.ts:25 |
| OpenRouter login / exchange timeout | 5 min / 30 s | packages/ai/src/auth/oauth/openrouter.ts:21-22 |
| Copilot policy retry | `500·2^n`, {maxRetries 2, maxElapsedMs 5000} | packages/ai/src/auth/oauth/github-copilot.ts:155,398 |
| auth.json sync lock | 10 × 20 ms | packages/coding-agent/src/core/auth-storage.ts:70-71 |
| auth.json async lock | stale 30 s, deadline 30 s, backoff ≤1 s ×(1+rand) | packages/coding-agent/src/core/auth-storage.ts:120-121,144 |
| auth.json mode / dir mode | 0o600 / 0o700 | packages/coding-agent/src/core/auth-storage.ts:25,59 |

## Evolution
- 2025-12-19 `1167e8445` resolve API key per LLM call (#223).
- 2025-12-28 `c10fc1e08` WebCrypto PKCE.
- 2026-01-03 `17b3a14bf` /model availability without refresh; 2026-01-05 `6a8609ac5` lock + re-read + re-check under lock (#466 silent logout); 2026-01-06 `d2f3b42de` refresh failure returns undefined so app starts.
- 2026-01-09 `f5e6bcac1` remove Anthropic OAuth → `19b566334` revert.
- 2026-02-06 `ddd5a65c7` compromised lock; 2026-03-06 `58f8fcd8f` sync lock retries (#1871).
- 2026-03-13 `92882dc4c` Anthropic flow update (platform.claude.com, port); 2026-03-14 `c7309aeda` fetch; 2026-03-15 `e645995a4` manual callback redirect_uri + refresh scope.
- 2026-04-20 `ebc60aa82` bind host override; 2026-05-28 `9d5fb70b7` Codex device login.
- 2026-06-03 `135fb545f` auth file mode on creation; 2026-06-10 `f63095cff` provider-owned auth + `CredentialStore.modify`.
- 2026-07-01 `e2ccdc850` Copilot first-poll delay; `f8bec25f3` surface save failures; 2026-07-03 `8133c94db` honor server slow_down; 2026-07-16 `6442536b1` OAuth flows bundled in Bun binaries; 2026-07-24 `a9f5b1c12` Radius OAuth via gateway; 2026-07-27 `99e34013d` auth print; `61da9e2f3` OpenRouter paste fallback.
- 2026-07-31..08-05 `57cde8690`, `d2be68dbe`, `2c79ce453`, `7cf90c1d1`, `4d68d9355` credential-file read path.
- 2026-08-04 `acbdc0d25` refresh bounded by 15 s (#7508); 2026-08-15/19 `d5278eaac`, `55b0db4d3` Copilot login rate limits.
- 2026-09-20 `b73412a37` Meta Muse; 2026-09-29 `02eed88fd` Sign in with ChatGPT; `4df157433` shared callback server; 2026-09-30 `7a11fe1c7` Anthropic copy-code login.
- 2026-10-02 `eeac84ca9` ChatGPT port conflict hard-fail; 2026-10-04 `bde882c74` refresh non-cancellable once started; 2026-10-07 `8d8ae2fc2` Anthropic free-port fallback.

## Evidence commits
1167e8445, c10fc1e08, 17b3a14bf, 6a8609ac5, d2f3b42de, f5e6bcac1, 19b566334, ddd5a65c7, 58f8fcd8f, 92882dc4c, c7309aeda, e645995a4, ebc60aa82, 9d5fb70b7, 135fb545f, f63095cff, e2ccdc850, f8bec25f3, 8133c94db, 6442536b1, a9f5b1c12, 99e34013d, 61da9e2f3, 57cde8690, d2be68dbe, 2c79ce453, 7cf90c1d1, 4d68d9355, acbdc0d25, d5278eaac, 55b0db4d3, b73412a37, 02eed88fd, 4df157433, 7a11fe1c7, eeac84ca9, bde882c74, 8d8ae2fc2, fe66edd94, b0bd0ff9d

## Quirks
- **Standalone Bun binary embeds OAuth flows statically** instead of lazy imports: `registerBunOAuthFlows()` registers anthropic, openaiCodex, openaiChatGPT, githubCopilot, openrouter, kimiCoding, meta, xai, radius (`packages/ai/src/bun-oauth.ts:13-25`).
- Browser callback pages are self-rendered HTML with escaping (`oauthSuccessHtml`/`oauthErrorHtml`, `packages/ai/src/utils/oauth-page.ts:3,94,102`).
- Legacy extension OAuth prompt types kept for old extensions (`OAuthPrompt`, `packages/ai/src/compat/extension-oauth-types.ts:1-8`).
- Anthropic reuses PKCE verifier as `state` → verifier appears in authorize URL; security impact unclear (unverified; mirrors Claude Code?).
- `lazyOAuth` memoizes the load promise (`promise ??=`, `auth/helpers.ts:46-50`) — a rejected import is cached for the process lifetime (inferred, untested; unverified).
- Codex token stored with no expiry margin while others use 1–5 min.
- Client ids of first-party CLIs embedded base64-obfuscated (Anthropic, Copilot) → see [[provider-identity-shim]].
- MCP OAuth has its own refresh lock (stale 20 s / wait 25 s / retry 100 ms; skew 30 s) (`packages/coding-agent/src/extensions/mcp/oauth.ts:46-53`) — separate from this path ([[mcp-integration]]).

## Failures
- [[oauth-refresh-token-rotation-lost]]
- [[credential-refresh-on-availability-path]]
- [[credential-file-lock-contention]]
- [[oauth-loopback-callback-failures]]
- [[device-code-polling-hang]]
- [[login-side-effects-rate-limited]]
- [[credential-expires-mid-run]]
