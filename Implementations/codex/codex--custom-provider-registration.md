---
type: implementation
harness: codex
concept: custom-provider-registration
commit: 622e9e3696
files: [codex-rs/model-provider-info/src/lib.rs:41, codex-rs/model-provider-info/src/lib.rs:96-102, codex-rs/model-provider-info/src/lib.rs:134-210, codex-rs/model-provider-info/src/lib.rs:300-380, codex-rs/model-provider-info/src/lib.rs:426-460, codex-rs/model-provider-info/src/lib.rs:477-497, codex-rs/model-provider-info/src/lib.rs:526-567, codex-rs/model-provider-info/src/lib.rs:653-755, codex-rs/model-provider-info/src/capabilities.rs:12-28, codex-rs/protocol/src/config_types.rs:564-592, codex-rs/ollama/src/lib.rs:15-53, codex-rs/ollama/src/client.rs:29-30, codex-rs/lmstudio/src/lib.rs:7-45, codex-rs/lmstudio/src/client.rs:15-16]
---
[[custom-provider-registration]] in [[codex]]. The registry is config-only: no plugin API, no custom stream functions.

## Mechanism
- **Declaration.** Providers live in `config.toml [model_providers.<id>]` (`ModelProviderInfo`, `codex-rs/model-provider-info/src/lib.rs:134-210`). Fields:
  - identity and endpoint: `name`, `base_url`, `model_catalog_url`, `wire_api` (Responses only), `query_params` (Azure `api-version`);
  - credentials: `env_key` with `env_key_instructions`, `experimental_bearer_token` ("discouraged in favor of `env_key` for security reasons"), `auth` (command auth), `gateway_oauth`, `aws`, `requires_openai_auth`;
  - headers: `http_headers`, `env_http_headers`;
  - transport tuning: `request_max_retries`, `stream_max_retries`, `stream_idle_timeout_ms`, `websocket_connect_timeout_ms`, `supports_websockets`, `supports_standalone_web_search`;
  - `capabilities {external_web_access, remote_compaction: unsupported|v2}` (`codex-rs/model-provider-info/src/capabilities.rs:12-28`);
  - `include_internal_metadata`.
- **Validation.** Conflicting auth options are rejected, e.g. `aws` with `env_key` or `experimental_bearer_token`, and `aws` with WebSockets (`codex-rs/model-provider-info/src/lib.rs:300-380`).
- **Command auth** `ModelProviderAuthInfo {command, args (redacted), timeout_ms, refresh_interval_ms, cwd}`. Defaults: `DEFAULT_PROVIDER_AUTH_TIMEOUT_MS` = 5_000 and `DEFAULT_PROVIDER_AUTH_REFRESH_INTERVAL_MS` = 300_000. A refresh interval of 0 means "only rerun after a 401 retry path" (`codex-rs/protocol/src/config_types.rs:564-592`).
- **Built-ins**: `openai`, `amazon-bedrock`, `amazon-bedrock-runtime`, `ollama` (port 11434), `lmstudio` (port 1234) (`codex-rs/model-provider-info/src/lib.rs:653-690`).
  - Policy comment: "We do not want to be in the business of adjucating which third-party providers are bundled with Codex CLI" (`codex-rs/model-provider-info/src/lib.rs:667-670`).
  - Built-ins cannot be overridden, except the Bedrock endpoint/auth/headers and the OpenAI `base_url` (`codex-rs/model-provider-info/src/lib.rs:695-720`).
- **OpenAI built-in** (`codex-rs/model-provider-info/src/lib.rs:526-567`):
  - header `version: <crate version>`;
  - env headers `OpenAI-Organization ← OPENAI_ORGANIZATION` and `OpenAI-Project ← OPENAI_PROJECT`;
  - WebSockets on.
  - A managed residency requirement adds `x-openai-internal-codex-residency: us` (`codex-rs/model-provider-info/src/lib.rs:41,447-452`; `bee042d119` 2026-09-16).
- **Base URL by auth.** ChatGPT-family auth routes to `https://chatgpt.com/backend-api/codex`; otherwise `https://api.openai.com/v1` (`codex-rs/model-provider-info/src/lib.rs:426-440`).
- **Default retry policy** for OpenAI-style providers: `retry_429: false`, `retry_5xx: true`, `retry_transport: true`, base delay 200 ms (`codex-rs/model-provider-info/src/lib.rs:454-460`).
- **Built-in `openai` override**: top-level `openai_base_url` (non-empty) rebuilds the built-in provider before user `model_providers` are merged; also handed to the credential broker (`codex-rs/core/src/config/mod.rs:3834-3840`, `:3751`; `codex-rs/model-provider-info/src/lib.rs:661-670`).
- **OSS providers (`--oss`)**: — default chosen by `oss_provider` (`"lmstudio"` | `"ollama"`, validated, persisted by `set_default_oss_provider`, `codex-rs/core/src/config/mod.rs:2444-2449`).
  - Base URL is `http://localhost:{CODEX_OSS_PORT or default}/v1`, or `CODEX_OSS_BASE_URL` (experimental) (`codex-rs/model-provider-info/src/lib.rs:736-755`).
  - Ollama: default model `gpt-oss:20b`; probe `/api/tags` or `/v1/models` with a 5 s timeout; auto-pull with progress; Ollama ≥ 0.13.4 required for Responses (`codex-rs/ollama/src/lib.rs:15-53`; `codex-rs/ollama/src/client.rs:29-30,98-110`).
  - LM Studio: default `openai/gpt-oss-20b`; download through the `lms` CLI and a background load (`codex-rs/lmstudio/src/lib.rs:7-45`; `codex-rs/lmstudio/src/client.rs:15-16`).
- **Bedrock**:
  - Mantle endpoint (Responses-compatible) with `x-amzn-mantle-client-agent: codex` (`codex-rs/model-provider-info/src/lib.rs:96-99`).
  - SigV4 signing via `codex-rs/aws-auth` (`1cd3ad1f49`).
  - Its own catalog (`codex-rs/model-provider/src/amazon_bedrock/catalog.rs`).
  - Unbounded connection retries disabled (`codex-rs/core/src/responses_retry.rs:100`).
  - The service tier must be advertised (`codex-rs/core/src/client.rs:979-988`).
- **Gateway OAuth** for custom providers (`codex-rs/model-provider-info/src/gateway_oauth.rs`; refresh skew 30 s, HTTP timeout 20 s, browser timeout 180 s: `codex-rs/login/src/gateway_auth.rs:54-55`, `codex-rs/login/src/gateway_auth_callback.rs:15`).

## Constants
| name | value | path:line |
|---|---|---|
| OSS default ports | Ollama 11434, LM Studio 1234 | `codex-rs/model-provider-info/src/lib.rs:653-654` |
| Ollama minimum version for Responses | 0.13.4 | `codex-rs/ollama/src/lib.rs:46-48` |
| local server probe timeout | 5 s | `codex-rs/ollama/src/client.rs:30`; `codex-rs/lmstudio/src/client.rs:16` |
| command-auth timeout / refresh | 5_000 ms / 300_000 ms | `codex-rs/protocol/src/config_types.rs:564-565` |

## Evolution
- `fe03320791` 2026-01-13 (#8798): Ollama built-ins switched to Responses.
- `d2394a2494` 2026-02-03: Chat Completions removed. `wire_api = "chat"` and `ollama-chat` now fail with a migration message.
- `a803790a10` 2026-04-16: provider runtime abstraction.
- `cefcfe43b9` 2026-04-20: Amazon Bedrock built-in. `1cd3ad1f49`: AWS SigV4 auth for OpenAI-compatible providers.
- `d19de6d150` 2026-04-24: Bedrock reasoning levels. `0db6811b7c` 2026-04-24: function `apply_patch` for Bedrock, superseded when the function variant was deleted (`e783341b70` 2026-05-08). `966932124c` 2026-05-30: Bedrock GPT models limited to the default service tier.
- `0cc80b8db5` 2026-08-20: turn cost for custom providers.
- `bee042d119` 2026-09-16: residency header.

## Versus pi
- [[pi--custom-provider-registration|pi]] offers `pi.registerProvider` (code) and `models.json` (declarative), and plugins can bring their own `streamSimple`.
- Codex providers are declarative only and must speak Responses; there is no runtime extension point for new wire formats ([[no-plugin-providers]]). See [[provider-breadth]].

## Failures
- [[endpoint-rejects-request-field]]
