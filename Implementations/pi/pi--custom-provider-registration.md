---
type: implementation
harness: pi
concept: custom-provider-registration
commit: b30a6dd77
files: [packages/coding-agent/src/core/model-registry.ts:210, packages/coding-agent/src/core/provider-composer.ts:40, packages/coding-agent/src/core/model-runtime.ts:921, packages/coding-agent/docs/custom-provider.md:19, packages/ai/src/models.ts:1041, packages/ai/src/api/pi-messages.ts:1, packages/coding-agent/src/core/sdk.ts:382]
---
[[custom-provider-registration]] in [[pi]].

## Mechanism
- **Two layers of "custom provider"**:
  - pi-ai: `createProvider({id, baseUrl, auth, models, api})` — Groq is 15 lines (`packages/ai/src/providers/groq.ts:6-14`); `Api = KnownApi | (string & {})` open set (`packages/ai/src/types.ts:29`); API impl module exports `stream` + `streamSimple` (+ optional `fetchDeferred/cancelDeferred`) = `ProviderStreams` (`packages/ai/src/types.ts:292-305`). Extensions build streams with exported `createAssistantMessageEventStream()` (`packages/ai/src/utils/event-stream.ts:132-135`).
  - coding-agent: `pi.registerProvider(name, ProviderConfigInput)` (legacy) or a native pi-ai `Provider` (`packages/coding-agent/src/core/model-registry.ts:210-219`; `docs/custom-provider.md:19-30`); `unregisterProvider` (`packages/coding-agent/src/core/extensions/types.ts:1844`). Registrations queued until the runner binds, immediate afterwards (`packages/coding-agent/src/core/extensions/types.ts:1828-1829`, `packages/coding-agent/src/core/extensions/loader.ts:211-216`; `975de88eb`); transactional extension load discards providers on factory throw (`packages/coding-agent/src/core/extensions/loader.ts:244-268, 613-632`; `a69bef789`).
- **Legacy config** fields: `name, baseUrl, apiKey, api, streamSimple, images, classifiers, headers, authHeader, oauth{login, refreshToken, getApiKey, modifyModels, isSubscription}, models, refreshModels` (`provider-composer.ts:40-50, 91-108`); `streamSimple` requires `api` (`:535-537`). OAuth adaptor maps legacy callbacks `onAuth/onDeviceCode/onPrompt/onProgress/onManualCodeInput/onSelect` → unified `notify/prompt` (`:353-372`). `authHeader:true` injects `Authorization: Bearer <key>`, requires key (`:374-386`); OAuth-only providers get no fabricated API-key login (`:426-427`).
- **Re-registration = patch**: merges defined values over previous, keeps undefined ("legacy ModelRegistry contract"), validates in isolation — broken re-registration throws without touching stored config (`model-runtime.ts:921-942`; `39a9784d2` #3651).
- **Composition** (see [[pi--model-catalog|model-catalog]]): legacy extension with `models` **replaces** list across ops; without, only baseUrl override; `modelOverrides` apply to extension providers too (`c6251a866` #6367); per-model baseUrl honoured (`ddb8ed0c7` #4063); stored creds satisfy custom providers without redundant apiKey (`ce6a67fc9` #5953). Stream dispatch: extension `streamSimple` if `model.api === extension.api` → base provider → global API registry (`provider-composer.ts:582-603`).
- `refreshModels` contracts: native provider publishes via `context.publish({update})`; legacy returns definitions and replaces the registration's live models (`docs/custom-provider.md:103-108`; `provider-composer.ts:613-635`).
- **Custom stream contract** (`docs/custom-provider.md:126-146`): `start` → balanced block events → single terminal `done|error`; aborts → `aborted`; honor `onPayload`/`onResponse`/`onProviderStreamEvent`. Recommended test matrix incl. cross-provider handoff, Unicode boundaries, auth refresh cancellation (`:156-169`). Overflow guidance: normalize unknown overflow messages to `context_length_exceeded`, never rewrite rate limits as overflow (`:152-154`; see [[context-overflow-detection]]).
- **Extension streaming**: `ModelRegistry.stream/streamSimple/complete/classify/generateImages` with request-time auth (`model-registry.ts:128-188`; `1f78cea7a` #9272/#8964).
- **Provider-boundary hooks** (wired in `packages/coding-agent/src/core/sdk.ts:382-409, 432-434`): `before_provider_request` (return value REPLACES payload, chained; ← pi-ai `onPayload`), `before_provider_headers` (mutate headers in place, `null` deletes; `244f1deaf` #6350), `after_provider_response` (status+headers before body; `d131fcd4b` #3128), `provider_stream_event` (each raw provider event pre-normalization, awaited in order — slow handler delays stream; `002fc8385` #9901) (`packages/coding-agent/src/core/extensions/types.ts:887-916`; `packages/coding-agent/src/core/extensions/runner.ts:1361-1418`; `docs/extensions.md:109-111`). Example `provider-payload.ts` logs/replaces payload; `debug-provider.ts` raw stream capture.
- **Config-only route**: models.json provider with `api` = any registered API (incl. `"pi-messages"`), `baseUrl`, `apiKey` (literal/`$ENV`/`!cmd`, [[pi--credential-resolution|credential-resolution]]), `headers`, `compat`, `models[]`.
- **pi-messages (harness-native wire protocol)** (`packages/ai/src/api/pi-messages.ts`): single `POST <baseUrl>/messages` with `{model, context, options:{temperature,maxTokens,reasoning,cacheRetention,sessionId,toolChoice}}`, SSE of serialized assistant events + terminal done/error (`:1-10, 370-402`); normalized `TranscriptContext` sent verbatim → all provider translation server-side; spoken by Radius, "any backend can implement it" (`:7-9`). Events mirror `AssistantMessageEvent` minus `partial`; `text_end/thinking_end` carry `contentSignature` (+`redacted`); `toolcall_start` carries `id`,`toolName`; `done/error` carry `usage, responseId, providerThinkingLevel, rewrite` (`:54-87`). Client rebuilds `partial` (`createEventConverter`, `:180-274`), usage taken wholesale (no local `calculateCost`). Server rewrite impact `{policyId, policyVersion, changed, tokenCountChange, messageCountChange, systemPromptChanged}` → diagnostic `pi_messages_rewrite` (`:44-51, 169-178`). `debug` → `?debug=1` (`:371-373`). Retention: option → legacy `PI_CACHE_RETENTION=long` → undefined (`:347-353`). HTTP errors → `PiMessagesResponseError` + diagnostic `pi_messages_response_failure` (url, status, body ≤8192) (`:98-156, 337-342`). No retries, no timeout (`:401`). Missing terminal → `"<provider> stream ended without a terminal event"` (`:423`).
- Docs evals exercise this: `custom-provider.docs.eval.ts` (implement NDJSON streaming provider from API doc fixture + live probe), `openai-provider.docs.eval.ts` (Acme OpenAI-compatible) (packages/evals, see [[harness-evals]]).
- **Built-in `llama.cpp` provider extension** (worked example of a dynamic-catalog provider; `packages/coding-agent/src/extensions/llama/`, `docs/llama-cpp.md`): targets the llama.cpp **router** (`llama-server` without `-m`, loads/unloads GGUF models on demand, `docs/llama-cpp.md:3-9`). Default URL `http://127.0.0.1:8080`, overridable via `/login llama.cpp` credential `env.LLAMA_BASE_URL` or `LLAMA_BASE_URL`/`LLAMA_API_KEY` env (`extensions/llama/provider.ts:25-35`; `docs/llama-cpp.md:54-64`).
  - Catalog = router `GET /models` filtered by `modelIsSelectable`: `loaded`, `sleeping` ("requests wake them automatically"), or `unloaded` presets only when router `models_autoload` is on (`provider.ts:39-58`). Chat models map to `api:"openai-completions"`, zero cost, `input` gains `image` iff `architecture.input_modalities` has it, compat `supportsDeveloperRole/Store/ReasoningEffort/StrictMode:false`, `maxTokensField:"max_tokens"`; `reasoning` + `thinkingFormat:"qwen-chat-template"` only if the loaded model's chat template contains `enable_thinking` (`provider.ts:127-158`) — unloaded/sleeping models are not probed since querying would load/wake them (`provider.ts:264-267`).
  - Context window precedence: runtime `meta.n_ctx` → `--ctx-size/-c/-ctx` in the model's launch args → cached value → `n_ctx_train` → 128000 (`provider.ts:60-79`).
  - Every model is also published as a **classifier** (`typesafe-system-one` for native decision models whose `output_modalities` include `decisions`, else `llama-cpp-classify`); decision-only models hidden from `/model` (`provider.ts:81-125`) → [[pi--structured-classifier-api|structured-classifier-api]].
  - `refreshModels` restores the persisted catalog first, then lists live and `persist`s `{models, checkedAt}` (`provider.ts:233-281`); `/llama` forces a live refresh **even under `PI_OFFLINE`** because it already contacted the server (`extensions/llama/index.ts:54-62`).
  - `/llama` command (`index.ts:183`): load/unload (asks before unloading others; "never silently unload", never deletes files), Esc-cancellable load/download, Hugging Face search + quantization pick (`docs/llama-cpp.md:76-87`). HF token lookup `HF_TOKEN` → `$HF_TOKEN_PATH` → `$HF_HOME/token` → `$XDG_CACHE_HOME/huggingface/token` → `~/.cache/huggingface/token` (`extensions/llama/huggingface.ts:46-61`); quantization parsed from file names by regex incl. `UD-`/`IQ`/`MXFP` (`huggingface.ts:6-8`).

## Constants
| name | value | path:line |
|---|---|---|
| pi-messages diagnostic body cap | 8192 | packages/ai/src/api/pi-messages.ts:121 |
| Custom model defaults | ctx 128000, max 16384 | packages/coding-agent/src/core/provider-composer.ts:242-243 |

## Evolution
- 2026-01-24 `c725135a7` API provider registry; `177c69440` custom providers via `streamSimple`.
- 2026-02-18 `975de88eb` flush `registerProvider` after bindCore; `unregisterProvider`.
- 2026-04-16 `d131fcd4b` `after_provider_response`.
- 2026-04-24 `39a9784d2` re-register keeps models (#3651).
- 2026-05-01 `ddb8ed0c7` registered model baseUrls honored.
- 2026-06-23 `ce6a67fc9` custom providers use stored auth.
- 2026-07-06 `244f1deaf` `before_provider_headers`.
- 2026-07-09 `c6251a866` modelOverrides for extension providers.
- 2026-09-07 `1f78cea7a` extensions stream from custom providers (#9272).
- 2026-09-23 `002fc8385` provider stream events exposed (#9901).

## Evidence commits
c725135a7, 177c69440, 975de88eb, a69bef789, d131fcd4b, 39a9784d2, ddb8ed0c7, ce6a67fc9, 244f1deaf, c6251a866, 1f78cea7a, 002fc8385

## Quirks
- pi-messages `createErrorEvent` builds a **fresh empty message** (`pi-messages.ts:323-345`) — streamed partial content is dropped on client-side failure, unlike every other adapter (likely bug, unverified intent). See [[errors-as-stream-events]].
- Radius server-side translation (signatures, caching) not in repo (unverified).
- Legacy `models` replace vs models.json upsert — two different merge semantics for "add models to a provider".

## Failures
- [[provider-reregistration-replaces-config]]
- [[credential-scoped-config-dropped]]
- [[placeholder-sent-as-api-key]]
