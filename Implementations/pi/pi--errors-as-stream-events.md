---
type: implementation
harness: pi
concept: errors-as-stream-events
commit: b30a6dd77
files: [packages/ai/src/types.ts:366, packages/ai/src/api/lazy.ts:4, packages/ai/src/utils/error-body.ts:16, packages/ai/src/utils/models-error.ts:3, packages/ai/src/utils/diagnostics.ts:10, packages/ai/src/api/anthropic-messages.ts:887, packages/ai/src/api/openai-completions.ts:707, packages/ai/src/api/bedrock-converse-stream.ts:365, packages/ai/src/api/google-generative-ai.ts:288, packages/ai/src/api/pi-messages.ts:323, packages/ai/src/api/system-one-shared.ts:125]
---
[[errors-as-stream-events]] in [[pi]].

## Mechanism
- **Contract** (`StreamFunction`, packages/ai/src/types.ts:366-381): receives normalized transcript; must return `AssistantMessageEventStream`; direct `streamSimple()` may throw synchronously only when auth is missing; once a stream is returned all request/model/runtime failures are encoded in it; termination = `AssistantMessage{stopReason:"error"|"aborted", errorMessage}`.
- Protocol allows `error` without `start` for setup failures (types.ts:773-787). Adapters push `start` only after HTTP response/headers (anthropic-messages.ts:657-658; openai-completions.ts:384; google-generative-ai.ts:102; Bedrock at `messageStart` bedrock-converse-stream.ts:300-304; Codex WS lazily on first event openai-codex-responses.ts:1479-1491).
- `lazyStream()` turns async setup failure (auth, dynamic import) into synthetic error message with zero usage + `error` + `end` (packages/ai/src/api/lazy.ts:4-62, 64-74). Azure endpoint/deployment resolution deliberately inside lazyStream so missing config = stream error not sync throw (packages/ai/src/providers/azure.ts:10-42).
- **Catch-block recipe** (every adapter): strip scratch fields from the *partial* content (`partialArgs`, `partialJson`, `customInput`, `streamIndex`, `index`; Bedrock `redactedChunks` flushed), `stopReason = signal.aborted ? "aborted" : "error"`, `errorMessage = formatProviderError(normalizeProviderError(err))`, push `{type:"error"}` with partial message — already-streamed text/thinking/tool calls preserved (openai-completions.ts:707-729; openai-responses.ts:214-231; anthropic-messages.ts:887-897; google-generative-ai.ts:288-298; mistral-conversations.ts:169-172). Scratch strip origin `e2b40dfc8` (#3078) → [[stream-scratch-state-persisted]].
- Bedrock flushes buffered `redactedChunks` → base64 `thinkingSignature` at block stop **and** every terminal path ("a stream can settle without stopping every block"; `Uint8Array` would serialize ~10×) (bedrock-converse-stream.ts:683-703, 344-351; d57e531f5).
- **Early usage capture**: Anthropic reads usage at `message_start` so aborted streams still have input counts (anthropic-messages.ts:680-684; bc8d994a7) → partial usage carried on the error message. Abort contract: aborted stream keeps partial content + partial usage (packages/ai/README.md:1128-1160).
- **Bill before parse**: System One sets usage before parsing answers because malformed answers were still billed (packages/ai/src/api/system-one-shared.ts:125-127) → [[billed-call-lost-on-parse-error]].
- **Error normalization** (packages/ai/src/utils/error-body.ts): `normalizeProviderError` probes status `statusCode` (Mistral) → `status` (openai, genai) → `$metadata.httpStatusCode` → `$response.statusCode` (Bedrock) (:56-67); body `body` string → `error` plain object (openai SDK) → `$response.body` (:69-92); only plain-prototype objects count as bodies (AWS class instance stringified to `{"_events":...}` had replaced the real message) (:98-117; 4523528b2); body cap `MAX_PROVIDER_ERROR_BODY_CHARS=4000` + `... [truncated N chars]` (:16, 137-140); `formatProviderError` → `"<prefix> (<status>): <body>"` only if message lacks body (:119-135). Motivation: proxies yielding `"403 status code (no body)"`/`"Unknown: UnknownError"` (:3-8; 62fad94f1 #5763).
- Anthropic adapter does **not** import `normalizeProviderError` (relies on SDK `APIError.message`) — observation.
- **Stable error categories** (Bedrock): `formatBedrockError` uses human prefixes `Internal server error|Model stream error|Validation error|Throttling error|Service unavailable` because agent retry matches `server.?error`/`service.?unavailable` and overflow veto needs `^Throttling error` (bedrock-converse-stream.ts:365-410; a3bf1eb39, 62fad94f1); data-retention errors get AWS docs hint (0b6c95dda). Mistral `finish_reason:"error"` message gets "(server error)" so the regex retries it (mistral-conversations.ts:922-938; 7fb59f995 #10487). Responses/Azure prefix status into message (52e13870a #4232); provider name prefix (`${provider} API error`, 0c7bb7c5c; openai-responses.ts:34, 221-229) → [[error-text-breaks-retry-classification]].
- Friendly errors: `subscription_sharing_usage_limit_exceeded` appends "Check your ChatGPT usage: https://chatgpt.com/settings/usage" (openai-responses.ts:221-229); Codex usage-limit → "You have hit your ChatGPT usage limit (<plan> plan). Try again in ~N min." from `plan_type`/`resets_at` (openai-codex-responses.ts:1596-1621); OpenRouter appends `error.error.metadata.raw` (openai-completions.ts:720-727); Mistral `Mistral API error (<status>): <body ≤4000>` (mistral-conversations.ts:29, 269-286); Azure "Azure OpenAI API error" (azure-openai-responses.ts:21-23).
- **Diagnostics channel**: `AssistantMessageDiagnostic{type,timestamp,error{name,message,stack,code},details}` appended to message (packages/ai/src/utils/diagnostics.ts:10-47; types.ts:564). Instances: `bedrock_response_failure{status, errorCode (only names ending "Exception"), requestId}`, values >200 chars dropped not truncated ("a truncated request id is not a request id"), errorMessage kept byte-identical for retry classification, not on abort (bedrock-converse-stream.ts:412-458; 70bbe47a9); `anthropic_input_transformations` (server dropped stale thinking) (anthropic-messages.ts:871-883; 4e69b0c28); `provider_transport_failure` (Codex WS→SSE) (openai-codex-responses.ts:356-365); `pi_messages_response_failure` (pi-messages.ts:337-342), `pi_messages_rewrite` (:169-178).
- **ModelsError** codes `model_source, model_validation, provider, stream, auth, oauth`; message auto-appends cause because "Callers surface `error.message` only" (packages/ai/src/utils/models-error.ts:3-21). Unconfigured provider → `ModelsError("auth","Provider is not configured")` surfaced via lazyStream (packages/ai/src/models.ts:843-875).
- Post-loop validation turns protocol violations into errors: Anthropic aborted → throw, still `pending` → "Anthropic stream ended without a stop reason", error/aborted → throw errorMessage (anthropic-messages.ts:861-870); Google "stream ended without a finish reason" (google-generative-ai.ts:276-278); Responses "stream ended before a terminal response event" (openai-responses-shared.ts:760-762) → [[terminal-event-required]].
- Safety stops become errors with partial content: Google `Provider stopped with: <raw>` (google-generative-ai.ts:279-284); Anthropic refusal → error w/ explanation (eb1f87fa9) → [[stop-reason-mapping-gaps]].
- Non-chat ops: `classify()` never rejects — result `stopReason:"error"` (packages/ai/README.md:936); `generateImages()` failures as `stopReason:"error"` (packages/ai/README.md:818).
- Retry interplay: `retryAssistantCall` normalizes abort during backoff into `{...response, stopReason:"aborted"}` minus errorMessage (packages/ai/src/utils/retry.ts:229-236; 243f64be5) → [[auto-retry-backoff]].
- Coding-agent loop: on `error` event replaces partial with `response.result()` and emits `message_end`; persisted, skipped on replay (packages/agent/src/agent-loop.ts:443-456; transform-messages.ts:195-203) → [[partial-message-persistence]], [[transcript-replay-repair]].

## Constants
| name | value | path:line |
|---|---|---|
| MAX_PROVIDER_ERROR_BODY_CHARS | 4000 | packages/ai/src/utils/error-body.ts:16 |
| MAX_BEDROCK_DIAGNOSTIC_VALUE_CHARS | 200 | packages/ai/src/api/bedrock-converse-stream.ts:415 |
| pi-messages diagnostic body cap | 8192 | packages/ai/src/api/pi-messages.ts:121 |

## Evolution
- 2025-09-01 `bf1f410c2` partial results on abort.
- 2025-09-18 `2296dc405` aborted vs error split, `errorMessage`.
- 2025-10-26 `bc8d994a7` Anthropic usage captured at `message_start` so aborts keep token stats.
- 2026-03-30 `a3bf1eb39` Bedrock stable error prefixes (#2699).
- 2026-04-14 `e2b40dfc8` strip `partialJson` scratch from OpenAI Responses tool calls on finalize (#3078).
- 2026-05-18 `52e13870a` HTTP status prefixed into Responses/Azure messages (#4232).
- 2026-06-09 `0b6c95dda` Bedrock data-retention docs hint.
- 2026-06-17 `62fad94f1` `normalizeProviderError` surfaces HTTP body (#5763).
- 2026-06-23 `ef231c491` request-scoped auth inside stream setup.
- 2026-07-21 `243f64be5` aborted retry attempts reported unsuccessful.
- 2026-07-30 `70bbe47a9` structured Bedrock failure diagnostics (#7286).
- 2026-07-31 `4523528b2` only plain objects count as error bodies (#7205).
- 2026-09-02 `4e69b0c28` Anthropic input-transformation diagnostics.
- 2026-09-14 `0c7bb7c5c` Responses errors name the provider.
- 2026-09-29 `89a5c7bda` classifier usage + cost.
- 2026-10-07 `7fb59f995` Mistral `finish_reason:"error"` message made retryable (#10487).

## Evidence commits
bf1f410c2 · 2296dc405 · e2b40dfc8 · 62fad94f1 · 4523528b2 · 0b6c95dda · 70bbe47a9 · a3bf1eb39 · 52e13870a · 0c7bb7c5c · 7fb59f995 · bc8d994a7 · d57e531f5 · 243f64be5 · 89a5c7bda

## Quirks
- Retry/overflow classification is regex over `errorMessage`; `diagnostics` exist but classifiers ignore them (types.ts:564) — structured kinds would decouple adapters (open question).
- pi-messages `createErrorEvent` builds `content: []`, dropping streamed partial content (pi-messages.ts:323-345) — likely bug (unverified).
- Anthropic path lacks `normalizeProviderError` (no import) — proxy error bodies only via SDK message (unverified adequacy).

- Google `ApiError` has no headers, so `RetryInfo.retryDelay` in the body is never read → pure exponential backoff; body RetryInfo parsing existed only in the removed Gemini CLI provider (fd35d9188 2026-01-02 #370 → removed fe66edd94) (google-shared.ts:503-505).

## Failures
- [[provider-error-body-hidden]] · [[error-text-breaks-retry-classification]] · [[foreign-sdk-error-shape-skips-retry]] · [[stream-scratch-state-persisted]] · [[streamed-usage-misread]] · [[truncated-stream-accepted-as-success]]
