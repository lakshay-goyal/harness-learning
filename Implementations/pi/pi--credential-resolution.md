---
type: implementation
harness: pi
concept: credential-resolution
commit: b30a6dd77
files: [packages/ai/src/auth/resolve.ts:27, packages/ai/src/auth/helpers.ts:3, packages/ai/src/env-api-keys.ts:73, packages/ai/src/utils/provider-env.ts:5, packages/ai/src/models.ts:843, packages/coding-agent/src/core/runtime-credentials.ts:3, packages/coding-agent/src/core/provider-composer.ts:465, packages/coding-agent/src/core/resolve-config-value.ts:28, packages/coding-agent/src/core/model-runtime.ts:651, packages/ai/src/providers/anthropic.ts:28, packages/ai/src/providers/amazon-bedrock.ts:54, packages/ai/src/providers/cloudflare-auth.ts:10]
---
[[credential-resolution]] in [[pi]].

## Mechanism

### pi-ai core (`resolveProviderAuth`, `packages/ai/src/auth/resolve.ts:27-93`)
- `ProviderAuth{apiKey?: ApiKeyAuth, oauth?: OAuthAuth}`, ≥1 required; keyless/ambient providers still implement `apiKey.resolve()` to report "configured" (`auth/types.ts:247-255`; `models.ts:157-164`). `Credential = ApiKeyCredential{type:"api_key", key?, env?} | OAuthCredential` (`auth/types.ts:17-37`); `env` = provider-scoped config (e.g. `CLOUDFLARE_ACCOUNT_ID`) (`packages/ai/README.md:442-452`).
- Order: (1) explicit `overrides.apiKey` (request option) resolved through provider apiKey auth (`:56-68`); (2) **stored credential owns the provider** — oauth → `resolveStoredOAuth`, api_key → `apiKey.resolve` with stored key + env overlay; stored credential of a type without handler → `undefined`, no fallback (`:70-87`); (3) ambient env / AWS profile / ADC via `apiKey.resolve` with no credential (`:89-92`). "No silent env fallback after a failed refresh or for a credential type without a matching handler" (`:27-32`; `packages/ai/README.md:440`).
- `envApiKeyAuth(name, envVars)`: stored key wins, else first set env var; `source` = env var name or "stored credential" (`auth/helpers.ts:3-31`). Whitespace-only env treated unset (`auth/context.ts:25-28`).
- **Env lookup** (`utils/provider-env.ts:41-52`): provider-scoped `env` override → `process.env` → Bun fallback `/proc/self/environ` (Bun compiled binaries may expose empty `process.env` in Linux sandboxes, bun #27802; `:5-39`; `5b8deef2f` #3801). Provider-scoped env overrides `7f29e7a36`.
- **Request assembly** `applyAuth` (`packages/ai/src/models.ts:843-875`): `getAuth(model,{apiKey,env,signal})`; unconfigured → `ModelsError("auth","Provider is not configured")` surfaced as stream error via `lazyStream`. Field precedence: explicit `options.apiKey` > resolved; headers provider-auth → `model.headers` → `options.headers` → `transformHeaders`; **`auth.baseUrl` overrides `model.baseUrl`** (credential-derived endpoint, Copilot enterprise / business) (`:865-871`). Header merge case-insensitive; `null` suppresses a lower default (`packages/ai/src/types.ts:128,170`; `utils/headers.ts:11-23`).
- Direct `streamSimple` may throw synchronously when auth missing; adapters `assertRequestAuth` / "No API key for provider" unless an `authorization`/`x-api-key`/`cf-aig-authorization` header is present (dummy `"unused"` key then) (`anthropic-messages.ts:316-327`; `openai-completions.ts:87-91`; `7921ae499` "Require explicit provider API keys").
- `ModelsError` codes `model_source, model_validation, provider, stream, auth, oauth`; message auto-appends cause since "Callers surface `error.message` only" (`utils/models-error.ts:3-21`).

### Env-var map (`getApiKeyEnvVars`, `packages/ai/src/env-api-keys.ts:73-127`)
| provider | env / ambient | note |
|---|---|---|
| anthropic | `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_OAUTH_TOKEN`, `ANTHROPIC_API_KEY` | AUTH_TOKEN counts for discovery but `getEnvApiKey` skips it (must be `Authorization: Bearer`) (`:78-81, 156`); federation vars `ANTHROPIC_FEDERATION_RULE_ID`, `ANTHROPIC_ORGANIZATION_ID`, `ANTHROPIC_SERVICE_ACCOUNT_ID`, `ANTHROPIC_IDENTITY_TOKEN_FILE`, `ANTHROPIC_WORKSPACE_ID` (`:29-36`) |
| github-copilot | `COPILOT_GITHUB_TOKEN` only | generic `GH_TOKEN/GITHUB_TOKEN` dropped (`a8af0b5e9` #4485; `:74-76`) |
| google | `GEMINI_API_KEY` | `providers/google.ts:10-11` |
| google-vertex | `GOOGLE_CLOUD_API_KEY` or ADC (`GOOGLE_APPLICATION_CREDENTIALS` / `~/.config/gcloud/application_default_credentials.json`) + `GOOGLE_CLOUD_PROJECT|GCLOUD_PROJECT` + `GOOGLE_CLOUD_LOCATION` → sentinel `"<authenticated>"` | `:160-172`; ADC existence cached but **not while node modules still loading** (`:46-58`; `cf656c169` #1550) |
| amazon-bedrock | sentinel when `AWS_PROFILE` / key pair / `AWS_BEARER_TOKEN_BEDROCK` / ECS creds / `AWS_WEB_IDENTITY_TOKEN_FILE` | `:174-192` |
| qwen-token-plan-individual / moonshotai-cn / opencode(-go) | reuse `QWEN_TOKEN_PLAN_API_KEY` / `MOONSHOT_API_KEY` / `OPENCODE_API_KEY` | |
| huggingface / vercel-ai-gateway / radius / meta | `HF_TOKEN` / `AI_GATEWAY_API_KEY` / `RADIUS_API_KEY` / `META_API_KEY` | |
- Node builtins loaded via string-concatenated specifiers — "NEVER convert to top-level imports - breaks browser/Vite builds" (`env-api-keys.ts:1-24`).

### Per-provider resolvers
- **Anthropic** (`packages/ai/src/providers/anthropic.ts:28-70`): stored key (+stored env) → `ANTHROPIC_AUTH_TOKEN` as `headers.Authorization: Bearer` (not OAuth-shaped; `b256ac7d7` #4342 set SDK `authToken: null` so the Anthropic SDK no longer auto-reads it from env — scope to compatible providers (inferred)) → `ANTHROPIC_OAUTH_TOKEN` → `ANTHROPIC_API_KEY` → **workload identity federation** (rule id + org id + identity token file required; service account / workspace optional), returned as provider `env` with empty auth, last so keys win "as in the SDK" (`:49-69`; `a9424cd43` #10242). Federation client cached per `(baseUrl, federation config, fetch)` and cloned per request via `withOptions({defaultHeaders})` so the SDK's token exchange happens once (`anthropic-messages.ts:349-377, 1052-1066`); federation only for `provider === "anthropic"` and no key/auth header; `assertRequestAuth` skipped then (`:612-613, 933-935`). `PiAnthropic` disables SDK default credential chain (`:329-339`).
- **Bedrock** (`providers/amazon-bedrock.ts:54-79`): stored key → `AWS_BEARER_TOKEN_BEDROCK` → stored/ambient `AWS_PROFILE` → key pair → ECS `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI/FULL_URI` → `AWS_WEB_IDENTITY_TOKEN_FILE`; ambient creds detected, never copied into store (`:6-10`); login offers bearer / profile / existing chain (`:12-53`; `3ea064ea2`). Adapter: `options.profile || env.AWS_PROFILE` beats ambient keys; explicit `credentials` only from provider env keys when no profile, because SDK treats explicit credentials as overriding `config.profile` (`bedrock-converse-stream.ts:157-165, 215-218`; `b63403a50` #6957/#7176). Bearer = `options.bearerToken || options.apiKey || AWS_BEARER_TOKEN_BEDROCK` via SDK `config.token` + `authSchemePreference:["httpBearerAuth"]` (`:184-189, 240-243`; `454b9619c`). `AWS_BEDROCK_SKIP_AUTH=1` → dummy creds for unauthenticated proxies (`:183, 208-213`; `df527fb98`). Region: ARN region > `options.region`/`AWS_REGION`/`AWS_DEFAULT_REGION` > region parsed from explicit endpoint > `us-east-1` unless ambient profile exists (`:193-205, 1194-1201`; `8cef3c8d7`, `d4b473e29`, `5a8ea0bc8`); endpoint pinning in [[pi--http-transport-hardening|http-transport-hardening]].
- **Vertex** (`api/google-vertex.ts:99-103, 429-460`; `providers/google-vertex.ts:13-90`): API key only if non-empty, not marker `gcp-vertex-credentials`, not `<placeholder>` (`/^<[^>]+>$/`) (`3a13fa80c` #3221, `ff1ea1232` #2335); else ADC client with project (`options.project || GOOGLE_CLOUD_PROJECT || GCLOUD_PROJECT`) + location; missing → throw; service-account file via `GOOGLE_APPLICATION_CREDENTIALS` → `keyFilename`. Login offers api-key / ADC / service-account, stores project/location/credentials path in credential `env`; resolve: stored key → `GOOGLE_CLOUD_API_KEY` (`01f7faae9`) → ADC file; aborts checked between env reads. Ambient markers filtered in compat dispatch (`850c210b7`).
- **Cloudflare** (`providers/cloudflare-auth.ts:10-29, 87-101`): per-field merge credential → ambient env for `CLOUDFLARE_API_KEY`, `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_GATEWAY_ID` (`bdd5c53bc` #6021/#6292). Workers AI → `apiKey`; AI Gateway → `cf-aig-authorization: Bearer <key>` and **nulls** `Authorization`/`x-api-key` so SDK placeholder auth isn't treated as BYOK provider key by the gateway. Binding transport sentinel `cf-aig-authorization: Bearer cloudflare-gateway-binding` satisfies adapters' auth checks, stripped by gateway (`api/cloudflare-ai-binding.ts:47-57`).
- **Azure**: endpoint/deployment resolved inside `lazyStream` so unconfigured endpoints become stream errors (`providers/azure.ts:10-42`); base URL `options.azureBaseUrl > AZURE_OPENAI_BASE_URL > AZURE_OPENAI_RESOURCE_NAME → https://<r>.openai.azure.com/openai/v1 > model.baseUrl` (`azure-openai-config.ts:68-94`); apiVersion option > `AZURE_OPENAI_API_VERSION` > `v1`; deployment option > `AZURE_OPENAI_DEPLOYMENT_NAME_MAP` (`modelId=deployment,…`) > model.id (`:14-35, 96-107`). Provider renamed `azure-openai-responses → azure` (breaking for auth.json keys; `a37306d43`).
- **Codex**: Bearer JWT + `chatgpt-account-id` claim (`openai-codex-responses.ts:1627-1659`). **OpenAI**: Sign-in-with-ChatGPT detected as provider `openai`, baseUrl exactly `https://api.openai.com/v1`, key not starting `sk-` (`openai-responses.ts:40-47`).

### coding-agent effective order (`docs/models.md:23`)
1. Request `options.apiKey` (pi-ai). 2. **Runtime `--api-key`** — `RuntimeCredentials.read` overlays non-persistent override before store (`runtime-credentials.ts:3-28`; `packages/coding-agent/src/main.ts:828-837`). 3. **Stored auth.json** (owns provider). 4. **models.json / extension `apiKey`** when no credential; extension apiKey beats models.json (`provider-composer.ts:388-393, 465-470`). 5. Inherited builtin resolver: env / ambient cloud creds (`:471-473`).
- Auth status labels `runtime | stored | models_json_command | environment | models_json_key | fallback` (`model-runtime.ts:639-649`; `provider-composer.ts:110-114, 718-732`).
- `prepareRequest` (`model-runtime.ts:651-690`): unknown provider → `ModelsError("provider")`; `getAuth`; headers `mergeHeaders(auth.headers, options.headers)` case-insensitive (`:155-169`) then `transformHeaders`; env = auth env ∪ options env; credential `baseUrl` overrides model baseUrl (`:679`; `e741cb05c`); `getAuth` also merges model-level headers resolved against env (`:550-570`). Null header deletion markers preserved through `getApiKeyAndHeaders` (`model-registry.ts:33-41`; `a24fb9e96` #7030/#7539); no auth + `authHeader` → "No API key found" (`:89-118`).
- **Per-request resolution** in the loop: `getApiKey(provider)` per LLM call (`packages/agent/src/agent-loop.ts:381-407`; `1167e8445` #223).
- **Config value syntax** (`resolve-config-value.ts`): `!cmd` → shell stdout trimmed; `$NAME`/`${NAME}` interpolation; `$$`→`$`, `$!`→`!`; else literal (`:28-86, 138-151`). Env lookup provider env → process.env, empty = missing (`:88-90`). `execSync` 10 s timeout, stderr ignored; Windows tries configured shell (stdin) first (`:153-206`). **Two caching policies**: `resolveConfigValue` caches command output for process lifetime — auth.json keys (`:9-10, 208-216`; `auth-storage.ts:446`; `docs/providers.md:84`: empty/timeout/nonzero leaves key unresolved until restart); `resolveConfigValueUncached` for models.json/extension keys and headers — "run at request time and are not cached" (`provider-composer.ts:467, 477`; `:221-227`; `docs/models.md:64`; `7a786d88a` #1835). Errors name missing vars / failed command (`:229-251`). Stored `env` object overrides process env for that provider (`docs/providers.md:90-102`).
- MCP: provider-token auth (`"auth":{"provider":…}`) only in global config "so a repository cannot pick where the credential goes" (`extensions/mcp/config.ts:25-26, 133-135`).
- Evals sandbox: reads auth.json into `InMemoryCredentialStore`, deletes auth.json + credential env var inside sandbox after resolving (`packages/evals/src/harness.ts:312-335, 441-445`).
- **Credential export for other tools** — `pi auth` subcommand (`packages/coding-agent/src/cli/auth-command.ts:5-45`, run before agent start, `main.ts:133-206`): `print-api-key --provider|--model`, `print-bearer-token … [--min-expiry <dur>]` (OAuth access token for an external client), `check [--json] [--credentials] [--no-refresh]`. Printing deliberately goes through `ModelRuntime.getAuth()`, so an OAuth token with < 5 min left is **refreshed and persisted** first (`cli/credential-print.ts:10-16`); `check` refreshes by default unless `--no-refresh` (`cli/auth-check.ts:22-64`). Status → exit code `ready` 0 / `not_ready` 1 / `invalid` 2 (`main.ts:200-206`); reasons `provider_not_found | credentials_not_configured | credential_not_available | invalid_state` (`auth-check.ts:9-13`). Effectively turns pi's `auth.json` + OAuth logins into a credential helper for other CLIs → [[pi--subscription-oauth-auth|subscription-oauth-auth]].
- Missing-credential messages share one pointer to `docs/providers.md` + `docs/models.md` and `/login` (`core/auth-guidance.ts:6-25`).

## Constants
| name | value | path:line |
|---|---|---|
| Vertex/Bedrock ambient sentinel | `"<authenticated>"` | packages/ai/src/env-api-keys.ts:160-192 |
| Vertex placeholder regex | `/^<[^>]+>$/` | packages/ai/src/api/google-vertex.ts:99-103 |
| Config command timeout | 10 s | packages/coding-agent/src/core/resolve-config-value.ts:185-196 |
| Cloudflare binding sentinel | `Bearer cloudflare-gateway-binding` | packages/ai/src/api/cloudflare-ai-binding.ts:47-57 |
| Azure default apiVersion | v1 | packages/ai/src/api/azure-openai-config.ts:4 |

## Evolution
- 2025-12-10 `a248e2547` stop deleting `process.env.ANTHROPIC_API_KEY` for OAuth (#164).
- 2025-12-19 `1167e8445` per-call `getApiKey` (#223).
- 2026-01-18 `def9e4e9a` shell commands for models.json keys (#762); 2026-02-04 `9cf5758b6` commands/env in auth.json keys.
- 2026-02-06 `df527fb98` unauthenticated Bedrock proxies.
- 2026-02-25 `cf656c169` don't cache false for Vertex ADC during import race.
- 2026-03-09 `01f7faae9` `GOOGLE_CLOUD_API_KEY`; 2026-03-18 `ff1ea1232` ignore placeholder keys; 2026-03-27 `7a786d88a` models.json auth per request.
- 2026-04-15 `3a13fa80c` Vertex marker = ADC; 2026-04-18 `454b9619c` SDK token auth for Bedrock; 2026-04-27 `5b8deef2f` Bun `/proc/self/environ`.
- 2026-05-15 `a8af0b5e9` Copilot ignores generic GitHub tokens; 2026-05-17 `b256ac7d7` #4342; 2026-05-28 `3e9f71744` explicit `$VAR` (#5095); 2026-05-29 `7921ae499` require explicit keys.
- 2026-06-10 `f63095cff` provider-owned auth, stored credential owns provider; 2026-06-13 `9e9fc7947` uppercase plain values literal (#5661); 2026-06-16 `7f29e7a36` provider-scoped env.
- 2026-07-10 `3ea064ea2` Bedrock API key login; 2026-07-11 `bdd5c53bc` Cloudflare per-field merge; `850c210b7` ambient markers filtered; 2026-07-28 `b63403a50` Bedrock profile precedence.
- 2026-08-03 `a24fb9e96` preserve header deletion markers; 2026-08-04 `e741cb05c` preserve extension auth endpoints.
- 2026-09-30 `a9424cd43` Anthropic workload identity federation.

## Evidence commits
a248e2547, 1167e8445, def9e4e9a, 9cf5758b6, df527fb98, cf656c169, 01f7faae9, ff1ea1232, 7a786d88a, 3a13fa80c, 454b9619c, 5b8deef2f, a8af0b5e9, b256ac7d7, 3e9f71744, 7921ae499, f63095cff, 9e9fc7947, 7f29e7a36, 3ea064ea2, bdd5c53bc, 850c210b7, b63403a50, a24fb9e96, e741cb05c, a9424cd43, a37306d43

## Quirks
- auth.json `!command` cached for process lifetime (failures too) vs models.json uncached — asymmetry documented, rationale not (unverified).
- Sentinel strings (`"<authenticated>"`, `gcp-vertex-credentials`, `unused`, binding bearer) occupy the key slot — repeatedly leaked to providers before filtering.
- Bun `/proc/self/environ` fallback intentionally duplicated in pi-ai for direct consumers.

## Failures
- [[env-credential-discovery-misfires]]
- [[process-env-mutated-for-auth]]
- [[async-init-race-caches-wrong-value]]
- [[placeholder-sent-as-api-key]]
- [[config-value-indirection-ambiguity]]
- [[credential-scoped-config-dropped]]
- [[bedrock-credential-and-endpoint-precedence]]
- [[credential-expires-mid-run]]
- [[credential-file-lock-contention]]
