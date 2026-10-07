---
type: implementation
harness: pi
concept: unified-provider-api
commit: b30a6dd77
files: [packages/ai/src/types.ts:17, packages/ai/src/types.ts:292, packages/ai/src/types.ts:366, packages/ai/src/types.ts:773, packages/ai/src/models.ts:316, packages/ai/src/models.ts:877, packages/ai/src/utils/event-stream.ts:3, packages/ai/src/api/lazy.ts:4, packages/ai/src/utils/transcript.ts:30, packages/ai/src/providers/all.ts:136, packages/ai/src/api/anthropic-messages.ts:400, packages/ai/src/api/openai-responses-shared.ts:433, packages/ai/src/api/openai-completions.ts:558, packages/ai/src/api/google-generative-ai.ts:100, packages/ai/src/api/mistral-conversations.ts:491, packages/ai/src/api/pi-messages.ts:1]
---
[[unified-provider-api]] in [[pi]].

## Mechanism
**Package shape** — `@earendil-works/pi-ai` 1.0.4; pinned SDKs `@anthropic-ai/sdk 0.129.0`, `@aws-sdk/client-bedrock-runtime 3.1127.0`, `@google/genai 2.21.0`, `openai 7.19.0`, `partial-json 0.1.7`, `typebox 1.3.27` (packages/ai/package.json:69-80); Mistral + others raw fetch.
- Layers (packages/ai/README.md:237-370): API modules `src/api/<api-id>.ts` export `stream` + `streamSimple` (+ optional `fetchDeferred`/`cancelDeferred`) = `ProviderStreams` contract (packages/ai/src/types.ts:292-305) → lazy wrappers `src/api/<api-id>.lazy.ts` (`openAICompletionsApi = () => lazyApi(() => import("./openai-completions.ts"))`, packages/ai/src/api/openai-completions.lazy.ts:4) → provider factories `createProvider({id, baseUrl, auth, models, api})` (Groq = 15 lines, packages/ai/src/providers/groq.ts:6-14) → `createModels()`/`builtinModels()` (packages/ai/src/models.ts:992-994; packages/ai/src/providers/all.ts:184-190).
- `KnownApi` = 10 chat APIs: openai-completions, mistral-conversations, openai-responses, azure-openai-responses, openai-codex-responses, anthropic-messages, bedrock-converse-stream, google-generative-ai, google-vertex, pi-messages (packages/ai/src/types.ts:17-27); `Api = KnownApi | (string & {})` open set (types.ts:29). Separate op families: image `openrouter-images` (types.ts:31), classifiers (types.ts:35-39) → [[structured-classifier-api]]; model `type` chat|image|classifier, missing = chat (types.ts:1133-1138, 1181-1191).
- 42 built-in providers (packages/ai/src/providers/all.ts:136-181; `KnownProvider` types.ts:43-85), incl. regional/plan variants.
- **Provider → API map** (all `src/providers/<id>.ts`; env var via `envApiKeyAuth(label, [VAR])`, packages/ai/src/auth/helpers.ts):

| API | providers (env var / base URL, `providers/<id>.ts:line`) |
|---|---|
| openai-completions only | ant-ling `ANT_LING_API_KEY` (:10-13), baseten (:10-13), cerebras (:10-13), deepseek `api.deepseek.com` (:10-13), groq (:10-13), huggingface `HF_TOKEN` router (:10-13), moonshotai / moonshotai-cn (`.ai` / `.cn`, shared `MOONSHOT_API_KEY`, :10-13), nvidia NIM (:10-13), together (:10-13), qwen-token-plan / -individual / -cn (aliyuncs `token-plan.*/compatible-mode/v1`; plan + individual share URL and `QWEN_TOKEN_PLAN_API_KEY`, :10-13), xiaomi + xiaomi-token-plan-{ams,cn,sgp} (`*.xiaomimimo.com/v1`, :10-13), zai `api.z.ai/api/coding/paas/v4` / zai-coding-cn `open.bigmodel.cn` (:10-13), cloudflare-workers-ai (wrapped `cloudflareStreams`, cloudflare-workers-ai.ts:20) |
| anthropic-messages only | anthropic (:78-88; env `ANTHROPIC_AUTH_TOKEN`→Bearer header, then `ANTHROPIC_OAUTH_TOKEN`/`ANTHROPIC_API_KEY`, :34-43; env-api-keys.ts:29-36), **minimax / minimax-cn** (`api.minimax.io/anthropic`, `api.minimaxi.com/anthropic`, :10-13), **kimi-coding** `api.kimi.com/coding` (:11-22), **vercel-ai-gateway** `ai-gateway.vercel.sh` `AI_GATEWAY_API_KEY` (:11-14) |
| openai-responses only | openai (+ classifier `openai-decisions`, :13-33), **meta** `api.meta.ai/v1` (:11-22), **xai** `api.x.ai/v1` (:11-22) |
| multi-API (dispatch on `model.api`) | github-copilot {anthropic-messages, openai-completions, openai-responses} (:28-31); openrouter {anthropic-messages, openai-completions} + images `openrouter-images` + classifier `typesafe-system-one` served at `/api/v1/systemone` (:28-34); fireworks {anthropic-messages, openai-completions} `api.fireworks.ai/inference` (:11-16); opencode "OpenCode Zen" {anthropic, google-generative-ai, completions, responses} + `typesafe-system-one` (:19-26); opencode-go {anthropic, completions, responses} (:15-18); cloudflare-ai-gateway {anthropic, completions, responses} **pinned to all three** because models.dev "drops and restores `workers-ai/*` entries over time" (cloudflare-ai-gateway.ts:11-24); azure {azure-openai-responses, openai-completions} (azure.ts:50-52) |
| single native API | amazon-bedrock → bedrock-converse-stream (:88; auth chain stored key → `AWS_BEARER_TOKEN_BEDROCK` → `AWS_PROFILE` → access keys → ECS role → web identity, :55-77); google → google-generative-ai `GEMINI_API_KEY` (:10-13); google-vertex → google-vertex (`GOOGLE_CLOUD_API_KEY` or ADC `GOOGLE_APPLICATION_CREDENTIALS`, :71-74); mistral → mistral-conversations (:10-13); openai-codex → openai-codex-responses `chatgpt.com/backend-api`, OAuth only (:11-20); radius → pi-messages, gateway catalog **replaces** shipped baseline once known "Radius org owners can disable models" (radius.ts:25-40) |
| classifier-only | typesafe `TYPESAFE_API_KEY`, `typesafe-system-one` (typesafe.ts:8-18) → [[structured-classifier-api]] |
- `isSubscription: true` OAuth providers: anthropic, github-copilot, kimi-coding, meta, openai (Sign in with ChatGPT), openai-codex, xai (`grep isSubscription providers/*.ts`) → [[subscription-oauth-auth]].
- **Stream wrappers** (decorate `ProviderStreams`, keep adapters generic): `withOpenCodeSessionHeader` injects `x-opencode-session: <sessionId>` unless caller already set it (case-insensitive) (packages/ai/src/providers/opencode-headers.ts:3-25; `561a2e066` 2026-09-09); `cloudflareStreams`/`cloudflareClassifier` substitute `{CLOUDFLARE_ACCOUNT_ID}`/`{CLOUDFLARE_GATEWAY_ID}` placeholders from resolved provider env (packages/ai/src/providers/cloudflare-stream.ts:6-35; base URLs packages/ai/src/api/cloudflare.ts:1-19); `azureStreams` resolves endpoint + deployment *inside* `lazyStream` "so an unconfigured endpoint errors on the stream instead of throwing out of `stream()`" (packages/ai/src/providers/azure.ts:27-40; resolver packages/ai/src/api/azure-openai-config.ts:72) → [[errors-as-stream-events]].
- **Credential-dependent model filtering**: `Provider.filterAllModels(models, credential)` (packages/ai/src/models.ts:202, 730, 1016-1017); openai drops classifier models under OAuth because "Sign in with ChatGPT tokens only reach the Responses API; the Decisions API rejects them" (packages/ai/src/providers/openai.ts:27-29).
- Model-type guards `getModelType` (missing `type` = chat) / `isModelType` / `assertChatModel|ImageModel|ClassifierModel` throw `ModelsError("provider")` (packages/ai/src/utils/model-operations.ts:17-45); `image-models.ts` = compat reads of generated `IMAGE_MODELS` (new code: `Models.getModelOfType("image")`) (packages/ai/src/image-models.ts:1-8).
- `StringEnum()` typebox helper emits plain `{type:"string", enum}` instead of `anyOf/const` "compatible with Google's API and other providers that don't support anyOf/const patterns" (packages/ai/src/utils/typebox-helpers.ts:3-24) → [[strict-tool-schema-rejections]].
- Root barrel side-effect free: "no generated catalogs, no provider factories, no api-registry, no OAuth implementations, no compat" (packages/ai/src/index.ts:4-8); old global API at `@earendil-works/pi-ai/compat` (packages/ai/src/compat.ts:1-11, 224-232); deprecated `streamAnthropic` etc. (packages/ai/src/legacy-api-aliases.ts:28-37).
- Mixed-API providers dispatch on `model.api` (createProvider, packages/ai/src/models.ts:1041-1182); missing api → stream error "has no API implementation".

**Entry points** — `Models.stream/complete/streamSimple/completeSimple/streamDeferred/fetchDeferred/cancelDeferred/generateImages/classify` (packages/ai/src/models.ts:316-358). `stream()` takes `ApiStreamOptions<TApi>` (types.ts:263-282); `streamSimple()` provider-neutral `SimpleStreamOptions{reasoning, thinkingBudgets, toolChoice, deferred}` (types.ts:356-364). `complete* = stream*().result()` (models.ts:893-899, 910-916).
- Every request: `normalizeContext(context)` folds `systemPrompt`+`tools` into a leading `SystemMessage` → `lazyStream(model, async () => { requireChatProvider; applyAuth; provider.stream(...) })` (models.ts:877-908).
- `TranscriptContext` is a **branded type** only produced by `normalizeContext()` so "a raw `Context` cannot reach provider code by accident" (types.ts:759-771; packages/ai/src/utils/transcript.ts:30-34).
- Mid-conversation `SystemMessage{content, sections?, toolsAdded?, toolsRemoved?}` (types.ts:518-545); `resolveTranscript(ctx, supportsMidConvoSystemMessages)` keeps in place or `collapseSystemMessages` (transcript.ts:104-120); Responses-family mid-convo tool additions via `additional_tools` (e47b8e37a 2026-08-07) or synthetic tool_search pair; rendered as `Updated system prompt section "name":` (packages/ai/src/utils/text.ts:23-40) — see [[transcript-carried-system-prompt]].

**Event protocol** — `AssistantMessageEvent` union (types.ts:788-804):

| event | payload |
|---|---|
| start | partial |
| text_start / text_delta / text_end(content) | contentIndex, partial |
| thinking_start / thinking_delta / thinking_end(content) | contentIndex, partial |
| toolcall_start / toolcall_delta(JSON fragment) / toolcall_end(toolCall) | contentIndex, partial |
| done | reason ∈ stop/length/toolUse/deferred, message |
| error | reason ∈ aborted/error, error: AssistantMessage (partial preserved) |

- Rules (types.ts:773-787): `start` before any update; setup failure may emit `error` without `start`; `partial` is a **shared live accumulator, not a snapshot**; text/thinking empty at `*_start`, grow via deltas until authoritative `*_end`; redacted thinking may be complete at start; tool-call args at `toolcall_start` provider-specific.
- `StopReason = pending|stop|length|toolUse|error|aborted|deferred` (types.ts:456); `pending` only in partials (packages/ai/README.md:1093).
- `AssistantMessage` (types.ts:552-583): `api, provider, model, responseModel` (concrete model when router/fallback differs, types.ts:558), `responseId`, `providerThinkingLevel`, `thinkingLevel`, `diagnostics[]`, `usage`, `stopReason`, `deferred`, `errorMessage`, `rawStopReason`, `endTurn` (debug), `timestamp`, `durationMs`.
- Blocks: `TextContent{text, textSignature?}` (legacy id string or `TextSignatureV1{v:1,id,phase?:"commentary"|"final_answer"}`, types.ts:395-405); `ThinkingContent{thinking, thinkingSignature?, redacted?}` (types.ts:407-415); `ToolCall{id,name,arguments:JsonObject, thoughtSignature?, namespace?}` (types.ts:423-431); `ImageContent` (types.ts:417-421).
- `ToolResultMessage{content, details (IsJsonCompatible-checked), usage (tool's own), nestedCalls (codemode, never sent), isError, durationMs}` (types.ts:585-624).

**EventStream** — push-based async iterable, **two-stack FIFO** queue (packages/ai/src/utils/event-stream.ts:3-23); `push()` ignored after done (:43-44); `result()` promise on completion predicate (:46-49); `AssistantMessageEventStream` completes on done|error (:102-111), stamps `durationMs` with monotonic `performance.now()` unless already timed or `timestamp < stream start` (deferred fetches stay untimed) (:91-130); `createAssistantMessageEventStream()` exported for extensions (:132-135).
- `lazyStream()` returns stream synchronously, runs auth + dynamic import behind it; failure → synthetic error message, zero usage, `error` + `end` (packages/ai/src/api/lazy.ts:4-62; import failures :64-74) → [[errors-as-stream-events]].
- Persistable frames: `AssistantMessageFrameEncoder` (packages/ai/src/utils/assistant-message-frame.ts:8-33, 139-326) — start frame `stopReason:"pending"` (:77-92), skips already-covered deltas via per-block `coveredChars` (:310-325), `toolcall_checkpoint` (:238-262), terminal events produce no frame (:152-158); pure reducer `reduceAssistantMessageFrames` validates order (:372-490). No non-test consumer in packages/* at HEAD (packages/ai/README.md:726-752) → [[partial-message-persistence]].

**Shared adapter skeleton** (OpenAI family; same shape elsewhere): stream + `output` with zero usage `stopReason:"pending"` (packages/ai/src/api/openai-completions.ts:309-329) → key resolution (explicit, or dummy `"unused"` if caller sent `authorization`/`cf-aig-authorization`, else throw) (:87-91) → build params → `onPayload` may replace (:366-369) → SDK `maxRetries:0` in `retryProviderRequest` (:370-382) → `onResponse` (:383) → only then `start` (:384) → `onProviderStreamEvent` per raw chunk (:559) → post-loop validation → `done` or throw → catch strips scratch + pushes `error` (:707-729). Hooks wire to extension events `before_provider_request`/`after_provider_response`/`provider_stream_event` (packages/coding-agent/src/core/sdk.ts:382-409; 002fc8385, d131fcd4b) → [[extension-event-hooks]].

**Per-adapter wire table**

| API | SSE/stream decoder | stop-reason map | notable |
|---|---|---|---|
| anthropic-messages | **owned** SSE (`asResponse()` + `iterateSseMessages`, CR/LF/CRLF, comments, multi-line data, trailing flush; packages/ai/src/api/anthropic-messages.ts:400-528, 650); event whitelist `message_start/delta/stop, content_block_start/delta/stop` (:391-398, 546-548; 3e7ffff18); `event: error` → throw (:542-544) | end_turn→stop, max_tokens→length, tool_use→toolUse, refusal→error w/ `stop_details.explanation` (eb1f87fa9), pause_turn→stop, stop_sequence→stop, `sensitive`→error (ee7c0a7d1 #978), other→throw (:1613-1639) | blocks found by provider `index` scratch; `signature_delta` silent (:737-782); text/thinking init with start-event content (59ad3dead); incomplete → "ended before message_stop" (:566-568; 83592bb2d) |
| bedrock-converse-stream | SDK event stream; `messageStart` must be assistant (:300-304); text/reasoning blocks lazy on first delta (:582-643) | end_turn/stop_sequence→stop; max_tokens/model_context_window_exceeded→length; tool_use→toolUse; else error `Provider stopped with:` (:1177-1192; f8f75544b) | in-stream exceptions thrown (:320-330); `messageStart` emits `start` |
| openai-completions | SDK chunks; null chunks skipped (:560; 7e2689ac1); text/thinking singletons + N tool calls (:685-687) | stop/end→stop; length; tool_calls/function_call→toolUse; content_filter/network_error/unknown→error "Provider finish_reason: X" (:1562-1586; a79ca4119, d914d1c19) | no finish_reason → throw unless `supportsFinishReason:false` infers (:688-705; 98ffad043, 2c3041242) |
| openai-responses (+azure, codex) | `processResponsesStream` slot table keyed by `output_index` (packages/ai/src/api/openai-responses-shared.ts:433-533; 8c9dbffa3 #6009) | completed→stop; incomplete+max_output_tokens→length; other incomplete→error; failed/cancelled→error; queued/in_progress→stop ("wonky"); exhaustive (:779-809); stop+toolCalls→toolUse (:594-596) | terminal event required (:760-762; cd95c2749); unfinished tool call → throw (:766-776; 1b2aa0ca0); `final_answer` phase provisional stop (:443-447); message item `null` content tolerated (2d597f021); `error` event `|| "Unknown error"` dead code (:745-746) |
| openai-codex-responses | WS JSON frames or SSE (zstd body) → `mapCodexEvents` normalizes done/completed/incomplete → reuse Responses processor (packages/ai/src/api/openai-codex-responses.ts:743-793) | as Responses; `end_turn` → `output.endTurn` (c3e7bc60a) | SSE parser flushes at EOF (:799-859; 64eeb82a4) → [[http-transport-hardening]] |
| google-generative-ai / google-vertex | `generateContentStream`, `candidates[0]` only (packages/ai/src/api/google-generative-ai.ts:100-299, :111) | STOP→stop (→toolUse only if stop), MAX_TOKENS→length, 16 safety/other reasons→error; exhaustive `never` (packages/ai/src/api/google-shared.ts:441-469; 5093641a5 #8059) | thinking iff `part.thought===true` (google-shared.ts:128-130; 4f757fbe2); function calls arrive whole → start/delta/end burst (google-generative-ai.ts:211-219); no finishReason → error (:276-278) |
| mistral-conversations | own SSE (all CR/LF combos, multi-line data, `[DONE]`; throws on non-`choices` JSON) (packages/ai/src/api/mistral-conversations.ts:491-511) | null/stop→stop; length/model_length→length; tool_calls→toolUse; `error`→"Provider stopped with: error (server error)" (:922-938; 7fb59f995) | empty deltas ignored (:633-636; 8930b9ec0); toolcall_end for all calls only at stream end (:745-760) |
| pi-messages | pi's own SSE of serialized events (packages/ai/src/api/pi-messages.ts:1-10, 370-402) | server-side | client rebuilds `partial` (`createEventConverter` :180-274); missing terminal → error (:423) |

**pi-messages (harness-native wire protocol)** — single `POST <baseUrl>/messages {model, context, options:{temperature,maxTokens,reasoning,cacheRetention,sessionId,toolChoice}}`; normalized `TranscriptContext` sent verbatim; all provider translation server-side (Radius gateway) (pi-messages.ts:1-10, 370-402). `PiMessagesEvent` mirrors events minus `partial`; `text_end/thinking_end` carry `contentSignature` (:54-87); usage wholesale from server; server prompt-rewrite impact → `pi_messages_rewrite` diagnostic (:44-51, 169-178); `?debug=1` (:371-373); no retries/timeout (:401).

**Stop-reason provenance** — `rawStopReason` preserved everywhere (926eb15c1 Anthropic, 637737ca7 Bedrock, fe1c9b6d5 completions, 23cb385b6 Google).

**Lazy loading** — Bedrock lazy wrapper imports via *variable specifier* so bundlers can't follow into AWS SDK, `.ts`→`.js` rewrite (packages/ai/src/api/bedrock-converse-stream.lazy.ts:4-13); `setBedrockProviderModule()` for Bun binary (:15-24; packages/ai/src/bedrock-provider.ts:1-6); OAuth loader same trick (packages/ai/src/auth/oauth/load.ts:1-80; 6442536b1); Node builtins via string-concatenated specifiers "NEVER convert to top-level imports" (packages/ai/src/env-api-keys.ts:1-24).

**Faux provider** — test double `fauxProvider()` with scripted queue, random 3–5 token chunks, simulated prompt cache per sessionId, deferred support (packages/ai/src/providers/faux.ts:24-30, 103-156, 245-285, 543-661; ef6af5ebb).

**Custom stream contract** (extensions): start → balanced block events → single terminal; aborts → aborted; honor `onPayload`/`onResponse`/`onProviderStreamEvent` (packages/coding-agent/docs/custom-provider.md:126-146) → [[custom-provider-registration]].

## Constants
| name | value | path:line |
|---|---|---|
| KnownApi count | 10 | packages/ai/src/types.ts:17-27 |
| built-in providers | 42 | packages/ai/src/providers/all.ts:136-181 |
| faux chunk size | 3–5 tokens | packages/ai/src/providers/faux.ts:29-30 |
| faux default model | `faux-1`, 128k ctx, 16384 max | packages/ai/src/providers/faux.ts:24-28 (id), 503-504 (ctx/max defaults) |
| pi-messages diagnostic body cap | 8192 | packages/ai/src/api/pi-messages.ts:121 |

## Evolution
- 2025-08-17 `f064ea0e1` unified AI package (OpenAI, Anthropic, Gemini).
- 2025-09-01 `bf1f410c2` partial results on abort (incremental block accumulation).
- 2025-09-02 `66cefb236` "Massive refactor of API": function-based, async generator, typed escape hatches; `004de3c9d` AsyncIterable streaming API.
- 2025-09-18 `2296dc405` `aborted` split from `error`; `errorMessage`; error event carries `reason`.
- 2026-01-24 `c725135a7`, `177c69440` API provider registry; custom providers via `streamSimple`; `createAssistantMessageEventStream()`.
- 2026-03-04 `e0754fdbb`, `668ebc094` browser-safe pi-ai (Bedrock lazy, OAuth to `/oauth`, no `Function` imports).
- 2026-04-21 `4b926a30a` → revert `fc9220d2d` → reapply `e58d631c8` owned Anthropic SSE parser.
- 2026-06-10 `f63095cff` (phase 1 Models runtime, provider-owned auth), `ba93da9a9` (phase 2 `src/providers/`→`src/api/`, `.lazy.ts`, `ProviderStreams`), `fec0c3d12` (phase 3 `createProvider`, per-provider `.models.ts`), `6e98573f2` (sync reads + explicit async `refreshModels` — "sync-or-async union invited latent sync assumptions"), `8a0903ebf` (phase 5 side-effect-free root barrel).
- 2026-06-23 `129eb460c`, `ef231c491` request-scoped auth resolved before provider calls.
- 2026-08-17 `a829de0a7` `AssistantMessageFrame` reducer.
- 2026-09-16 `9e05370b2` mid-conversation system messages → `TranscriptContext`.
- 2026-09-23 `002fc8385` raw provider stream events exposed to extensions; `a328aa89a` image + classifier models unified.

## Evidence commits
f064ea0e1 · bf1f410c2 · 66cefb236 · 2296dc405 · c725135a7 · 177c69440 · e0754fdbb · 668ebc094 · 4b926a30a · fc9220d2d · e58d631c8 · 3e7ffff18 · f63095cff · ba93da9a9 · fec0c3d12 · 6e98573f2 · 8a0903ebf · 8c9dbffa3 · b2602be77 · 3ba22ce17 · f284a2460 · 9e05370b2 · 002fc8385 · a328aa89a · 6442536b1 · ef6af5ebb

## Quirks
- `partial` is live and mutable — consumers that store it must copy; frame encoder exists precisely to persist without snapshots (assistant-message-frame.ts:310-325).
- `mapStopReasonString` (google-shared.ts:474-483) appears unused after Cloud Code Assist removal `fe66edd94` (unverified).
- Responses `error` event fallback `|| "Unknown error"` is dead code (openai-responses-shared.ts:745-746).
- pi-messages error event is a fresh empty message, dropping streamed partial content (pi-messages.ts:323-345) — unlike every other adapter; intent unverified.
- Frame encoder has no in-repo non-test consumer at HEAD (unverified intended consumer: SDK/durable).

## Failures
- [[stop-reason-mapping-gaps]] · [[streamed-tool-call-fragmentation]] · [[stream-delta-assembly-errors]] · [[sse-framing-errors]] · [[empty-payload-rejections]] · [[endpoint-rejects-request-field]] · [[assistant-content-shape-misread]] · [[missing-optional-fields-crash-replay]] · [[quadratic-event-queue-drain]] · [[private-fields-break-duck-typed-streams]] · [[node-only-imports-break-browser-bundle]] · [[truncated-stream-accepted-as-success]]
