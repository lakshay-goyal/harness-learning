---
type: implementation
harness: codex
concept: secret-handling
commit: 622e9e3696
files: [codex-rs/secrets/src/sanitizer.rs:4, codex-rs/secrets/src/local.rs:14, codex-rs/arg0/src/lib.rs:297, codex-rs/protocol/src/shell_environment.rs:13, codex-rs/protocol/src/shell_environment.rs:125, codex-rs/config/src/shell_environment_policy.rs:136, codex-rs/responses-api-proxy/src/read_api_key.rs:8, codex-rs/responses-api-proxy/src/lib.rs:163, codex-rs/network-proxy/src/credential_broker.rs:43, codex-rs/app-server-protocol/src/protocol/item_builders.rs:58]
---
[[secret-handling]] in [[codex]].

## Mechanism
- **At rest**: `age` scrypt-encrypted files `local.age`, `codex_auth.age`, `mcp_oauth.age`, `gateway_oauth.age`, `SECRETS_VERSION = 1`, passphrase kept in the OS keyring (`codex-rs/secrets/src/local.rs:14-44`; keyring account doc in `codex-rs/secrets/src/lib.rs`); `codex-rs/keyring-store/src/lib.rs` "Shared credential store abstraction for keyring-backed implementations".
- **Display redaction** (`codex-rs/secrets/src/sanitizer.rs:4-20`): `sk-[A-Za-z0-9]{20,}`, `\bAKIA[0-9A-Z]{16}\b`, `Bearer <16+ chars>`, `(api[_-]?key|token|secret|password)\s*[:=]\s*<8+ chars>` → `[REDACTED_SECRET]`. Applied to command strings in app-server item builders (`codex-rs/app-server-protocol/src/protocol/item_builders.rs:58`, `:156-182`; e.g. `git -c 'http.extraHeader=Authorization: Bearer [REDACTED_SECRET]' push`) and gateway auth tokens (`codex-rs/login/src/gateway_auth_token.rs:8`) — UI/protocol display, **not** model context. Credentials kept out of app-server logs (`79b7606803`).
- **Config injection guard**: `~/.codex/.env` loaded at startup but may not set any `CODEX_*` var — "Security: Do not allow `.env` files to create or modify any variables with names starting with `CODEX_`" (`ILLEGAL_ENV_VAR_PREFIX`, `codex-rs/arg0/src/lib.rs:297-323`).
- **Child-process env**:
  - Hard denylist `NON_INHERITABLE_ENV_VARS` (case-insensitive, applied after shell-env-policy overrides): `CODEX_EXEC_SERVER_NOISE_AUTH_TOKEN`, `NODE_REPL_AUTH_TOKEN`, `CODEX_GUARDIAN_DECISIONS_API_KEY`, `OPENAI_FEDERATION_RULE_ID`, `OPENAI_IDENTITY_TOKEN_FILE`, `OPENAI_WORKLOAD_IDENTITY_CONTEXT` — "Environment variables that model-reachable child processes must not inherit"; doc: "not a filesystem security boundary for the referenced identity-token file" (`codex-rs/protocol/src/shell_environment.rs:13-40`). See [[harness-credential-leaks-to-tools]].
  - User `shell_environment_policy`: inherit → default excludes `*KEY*`, `*SECRET*`, `*TOKEN*` → custom excludes → `set` → `include_only` (`codex-rs/protocol/src/shell_environment.rs:110-140`; doc `codex-rs/protocol/src/config_types.rs:232-239`). **Default `ignore_default_excludes = true`** — the KEY/SECRET/TOKEN filter is off unless the user opts in (`codex-rs/config/src/shell_environment_policy.rs:136`; flipped `9fb9ed6cea`).
  - Credential broker: real credentials held in proxy memory, children get dummy values, substituted on TLS to bound hosts ([[codex--egress-policy-proxy]]).
  - Shell snapshot / replay credential hardening (`ec512d2347`); Windows sandbox env transport via launcher-only chunks ([[codex--os-level-sandbox]]).
- **responses-api-proxy** (`codex-rs/responses-api-proxy/`): lets an unprivileged user or sandboxed agent use an API key it cannot read — privileged user pipes the key on stdin; proxy builds `Bearer <key>`, `mlock(2)`s the header memory and zeroizes buffers (`codex-rs/responses-api-proxy/src/read_api_key.rs:8-9`, `:39-141`); forwards only exact `POST /v1/responses` (no query) to `https://api.openai.com/v1/responses`, replacing any incoming Authorization; everything else 403 (`codex-rs/responses-api-proxy/src/lib.rs:53`, `:163-230`); optional `--dump-dir` with Authorization/cookie redaction; `--http-shutdown`. Process-hardened ([[codex--harness-process-hardening]]).
- **Error payloads**: catalog decode errors report only category/line/column/byte count; connection errors without URLs (`977193486d`, `5a0d0929e2`; `codex-rs/codex-api/src/sse/responses.rs:573-585`) — see [[error-diagnostics-echo-payload]].
- **User verification** (`codex-rs/user-verification/`): device credential (P-256, macOS Secure Enclave, biometric signing policy) signing challenges per account-user; no network registration; app-server RPC only (`codex-rs/app-server/src/user_verification.rs`), not wired into tool approvals (`b7ad941b1f`, `e7637306bc`).
- **Memory workspace**: symbolic links rejected (`a04940cb12`).
- Unrelated-looking but adjacent: `codex-rs/core/src/attestation.rs` produces an `x-oai-attestation` header just in time (`:7-25`) — client attestation, not secret storage.

## Constants
| name | value | path:line |
|---|---|---|
| redaction patterns | `sk-…{20,}`, `AKIA…{16}`, `Bearer …{16,}`, key/token/secret/password assignment {8,} | `codex-rs/secrets/src/sanitizer.rs:4-20` |
| .env forbidden prefix | `CODEX_` | `codex-rs/arg0/src/lib.rs:297` |
| non-inheritable env vars | 6 names | `codex-rs/protocol/src/shell_environment.rs:13-21` |
| default env excludes | `*KEY*`, `*SECRET*`, `*TOKEN*` (off by default) | `codex-rs/protocol/src/shell_environment.rs:125-130`; `codex-rs/config/src/shell_environment_policy.rs:136` |
| SECRETS_VERSION | 1 | `codex-rs/secrets/src/local.rs:14-44` |

## Evolution
- 2025-05-22 `cb379d7797` `shell_environment_policy` in config.
- 2025-09-26 `c549481513` responses-api-proxy; 2025-09-27 `68765214b3` its 30 s timeout removed (long streams).
- 2025-12-18 `9fb9ed6cea` default excludes off by default (`ignore_default_excludes = true`), `*SECRET*` added.
- 2026-06-24 `989f55defa` credential broker.
- 2026-08-08 `c4513cb982` launch context kept from children; 2026-08-17 `89e297729e` Noise auth tokens; 2026-08-18 `fe50b61689` Node REPL auth tokens.
- 2026-08-21 `79b7606803` credentials out of app-server logs.
- 2026-09-07/08 `b7ad941b1f` / `e7637306bc` user verification.
- 2026-09-09 `ec512d2347` shell snapshot / replay credential hardening.

## Versus pi
- [[pi--secret-handling]]: plaintext `auth.json` 0600 + `$ENV`/`!cmd` indirection + bug-report redaction; children inherit the full env. Codex: encrypted-at-rest with keyring passphrase, child-env denylist + optional pattern filter, proxy-held credentials, key-hiding proxy for unprivileged users.
