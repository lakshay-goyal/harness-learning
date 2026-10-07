---
type: implementation
harness: opencode
concept: errors-as-stream-events
commit: ecc4916b5a
files: [packages/opencode/src/session/message-v2.ts:619-747, packages/opencode/src/provider/error.ts:23-71, packages/opencode/src/provider/error.ts:102-193, packages/opencode/src/session/prompt.ts:1297-1307, packages/llm/src/route/executor.ts:233-273, packages/llm/src/protocols/anthropic-messages.ts:558-564]
---
[[errors-as-stream-events]] in [[opencode]].

## Mechanism

### Legacy runtime — errors become typed session errors on the assistant message
- `MessageV2.fromError` maps everything to a stored `error` on the assistant message: abort → `AbortedError`; ECONNRESET → retryable "Connection reset by server"; Bun `ZlibError` → retryable "Response decompression failed"; header timeout / SSE stall → retryable `ResponseStreamError`; `APICallError` → `ContextOverflowError` or `APIError {statusCode, isRetryable, responseHeaders, responseBody, metadata}`; `LoadAPIKeyError` → `AuthError`; length → `OutputLengthError`; unknown → `NamedError.Unknown` with JSON (`packages/opencode/src/session/message-v2.ts:619-747`).
- Message extraction prefers body `message`/`error`; HTML gateway pages become human text for 401/403 (`packages/opencode/src/provider/error.ts:57-67`).
- Mid-stream error objects → `parseStreamError` by code (`error.ts:102-139`).
- Non-success finishes get a visible terminal error: `content-filter` → `ContentFilterError` "The response was blocked by the provider's content filter" + `Session.Event.Error` (`packages/opencode/src/session/prompt.ts:1297-1307`).

### v2 runtime — typed reasons in `packages/llm`
- HTTP failures classified by reason before status: body `content[-_ ]?policy|content_filter|safety` → `ContentPolicy`; 401/403 → Authentication; 429 → `RateLimit` unless body says quota → `QuotaExceeded`; 400/404/409/413/422 → `InvalidRequest` with overflow classification; ≥500 → `ProviderInternal` (retryable) (`packages/llm/src/route/executor.ts:233-273`).
- Provider error events carry `code`, `type` and nested fields (`d0cb58782f`).
- Finish map: Anthropic `end_turn|stop_sequence|pause_turn` → `stop`, `refusal` → `content-filter`, unknown → `unknown` (`packages/llm/src/protocols/anthropic-messages.ts:558-564`). `pause_turn` (server-tool pause) is indistinguishable from stop — latent, since the v2 runner advertises no Anthropic server tools.
- Tool error values: structured JSON was stringified `[object Object]` (`130957288e`); defects dropped (`ba5c8d3822`); structured `message` getters (`c17b9557f1`).

## Constants
| name | value | path:line |
|---|---|---|
| provider error body cap (v2) | 16 384 chars, redacted then sliced | `packages/llm/src/route/executor.ts:35` |

## Evolution
- 2026-02-08 `62f38087b8` Responses-style mid-stream errors parsed (fatal codes stopped being retried).
- 2026-03-25 `7123aad5a8` `ZlibError` retryable.
- 2026-05-22 `d0cb58782f` v2 surfaces code/type/nested fields.
- 2026-06-11 `e2527db3c7` content-filter finish visible (Anthropic `refusal` used to leave a silent idle session).
- 2026-08-05 `709c195905` patched `@ai-sdk/openai-compatible` to keep structured error chunks.

## Quirks / drift
- A 5xx whose body mentions "safety" becomes non-retryable `ContentPolicy` in v2 (inference from ordering at `executor.ts:233-235`).

Failures: [[provider-error-body-hidden]] · [[stop-reason-mapping-gaps]] · [[deterministic-5xx-retried]].

Contrast: [[pi--errors-as-stream-events|pi]] ends the stream with `stopReason: error` + text and classifies by regex; opencode v2 classifies into typed reasons at the HTTP boundary.
