---
type: implementation
harness: codex
concept: credential-resolution
commit: 622e9e3696
files: [codex-rs/login/src/auth/manager.rs:954-969, codex-rs/login/src/auth/manager.rs:1489-1565, codex-rs/model-provider-info/src/lib.rs:149-157, codex-rs/model-provider-info/src/lib.rs:300-380, codex-rs/model-provider-info/src/lib.rs:477-497, codex-rs/protocol/src/config_types.rs:564-592, codex-rs/tui/src/onboarding/auth.rs:852, codex-rs/core/src/realtime_conversation.rs:1941]
---
[[credential-resolution]] in [[codex]].

## Mechanism
- **Load precedence** (`load_auth`, `codex-rs/login/src/auth/manager.rs:1489-1565`):
  1. `CODEX_API_KEY` env, if enabled for the caller (`codex-rs/login/src/auth/manager.rs:954,965`);
  2. ephemeral host-supplied tokens: "External ChatGPT auth tokens live in the in-memory (ephemeral) store. Always check this…";
  3. `CODEX_ACCESS_TOKEN` env, a PAT or an agent-identity JWT (`codex-rs/login/src/auth/manager.rs:955,969`);
  4. persisted storage (file, keyring or auto).

  "If the caller explicitly requested ephemeral auth, there is no persisted fallback" (`codex-rs/login/src/auth/manager.rs:1561`).
- **`OPENAI_API_KEY` is not a runtime credential for the main loop.** It is read only to prefill onboarding (`codex-rs/tui/src/onboarding/auth.rs:852`) and for realtime voice (`codex-rs/core/src/realtime_conversation.rs:1941`).
- **Per-provider credential sources** (custom providers, `codex-rs/model-provider-info/src/lib.rs:149-157,477-497`):
  - `env_key`: an env var name, with `env_key_instructions` shown when it is missing;
  - `experimental_bearer_token`: a literal, discouraged;
  - `auth`: a command whose stdout is the token, with a timeout of 5_000 ms and a cache refreshed every 300_000 ms. Refresh interval 0 means rerun only after a 401 (`codex-rs/protocol/src/config_types.rs:564-592`). Args are `RedactedString`;
  - `aws`: SigV4 (`codex-rs/aws-auth`);
  - `gateway_oauth`;
  - `env_http_headers`: header values taken from env vars.
- **Mutually exclusive sources are a config error**, not a precedence rule: `aws` cannot be combined with `env_key`, `experimental_bearer_token` or WebSockets (`codex-rs/model-provider-info/src/lib.rs:300-380`).
- **Workspace restrictions** (`forced_chatgpt_workspace_id`, allowed login methods) are enforced at load.
- **Admin pinning of auth**: `forced_login_method` = `chatgpt` | `api` restricts which login mechanism users may use; `forced_chatgpt_workspace_id` pins the ChatGPT workspace (`codex-rs/config/src/config_toml.rs:278-283`, enum `codex-rs/protocol/src/config_types.rs:559-562`; policy `codex-rs/config/src/auth_policy.rs:14`).

## Versus pi
- [[pi--credential-resolution|pi]]: request override > runtime flag > stored credential > config value (`$ENV`/`!cmd` sigils) > ambient env, with "stored credential owns provider".
- Codex: env API key > host tokens > env access token > stored, for the OpenAI/ChatGPT path. Custom providers name exactly one source, and conflicting sources are rejected at config time. No sigil syntax; command auth is a structured table with a timeout and a refresh interval.

## Failures
- [[harness-credential-leaks-to-tools]]
