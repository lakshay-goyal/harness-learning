---
type: implementation
harness: pi
concept: auto-retry-backoff
commit: b30a6dd77
files: [packages/ai/src/utils/retry.ts:7, packages/ai/src/utils/retry.ts:30, packages/ai/src/utils/retry.ts:128, packages/ai/src/utils/retry.ts:252, packages/ai/src/utils/provider-retry.ts:24, packages/coding-agent/src/core/agent-session.ts:3708, packages/coding-agent/src/core/agent-session.ts:3761, packages/coding-agent/src/core/settings-manager.ts:1023, packages/durable/src/harness/generation.ts:477]
---
[[auto-retry-backoff]] in [[pi]].

## Mechanism
**Two layers, one visible.**
- **Layer 1 — in-adapter SDK-equivalent retry** (`packages/ai/src/utils/provider-retry.ts`): SDK built-in retry timers ignore the request `AbortSignal`, so SDKs are invoked with `maxRetries: 0` and wrapped in `retryProviderRequest` with an abortable sleep (`provider-retry.ts:99-128`). Default `maxRetries = 0` → opt-in (`:111`). Retryable: `x-should-retry: true/false` header wins; no status (network) → retry; 408, 409, 429, ≥500 (`:24-37`, "mirrors pinned OpenAI/Anthropic SDK policy; review when either SDK is upgraded"). Delay: `retry-after-ms` → `retry-after` (seconds or HTTP-date) → `min(0.5·2^i, 8)s × (1 − rand·0.25)` jitter (`:53-69`). Server-requested delay > `maxRetryDelayMs` (default 60 000; 0 disables) → throw immediately `Server requested Ns retry delay (max: Ms)…` so the outer visible retry handles it (`:1`, `:39-51`); outer classifier matches `"retry delay"` (`retry.ts:90-92`, #1123). `noRetryStatuses` per-call opt-out (`:7-8,121`). Each retry is a fresh SDK request so `X-Stainless-Retry-Count` stays 0 (`:116`). Covers request → response headers only; mid-stream failures become error messages.
- **Layer 2 — agent-level message retry** in `AgentSession` with classifier in pi-ai `packages/ai/src/utils/retry.ts`:
  - `isRetryableAssistantError(msg)` (`retry.ts:252-257`): only `stopReason === "error"`; first veto `NON_RETRYABLE_PROVIDER_LIMIT_ERROR_PATTERN` (GoUsageLimitError, FreeUsageLimitError, "Monthly usage limit reached", "available balance", insufficient_quota, "out of budget", "quota exceeded", "billing", `subscription_sharing_usage_limit_exceeded`) (`:7-28`); then `RETRYABLE_PROVIDER_ERROR_PATTERN` (`:30-107`): overloaded, server_busy, "high demand", "model is at capacity", rate.?limit, 429, 500/502/503/504/520/524, service unavailable, internal error, "provider returned error", upstream buffer limit, network/connection refused/lost/"other side closed"/"fetch failed"/getaddrinfo/ENOTFOUND/EAI_AGAIN/"upstream connect"/"reset before headers"/socket hang up/timeout/"terminated", websocket closed/error, "ended without", "stream ended before message_stop", "stream ended before a terminal response event" (`:80-84`), "http2 request did not get a response", "pending stream has been canceled", "retry delay", "you can retry your request"/"try your request again"/"please retry your request", gRPC "ResourceExhausted", `subscription_sharing_usage_unavailable|user_unavailable`.
  - Session `_isRetryableError`: context overflow excluded ("handled by compaction") then classifier (`agent-session.ts:3708-3712`) → [[overflow-recovery]].
  - `_prepareRetry` (`agent-session.ts:3761-3801`): settings disabled → false; `++attempt`; `> maxRetries` → decrement ("preserve the completed attempt count") and give up; `delayMs = retryDelayMs(settings, attempt)`; emit `auto_retry_start{attempt,maxAttempts,delayMs,errorMessage}`; **omit failed attempt from model projection via `context_edit` null while keeping raw history** (`:3785-3786`, `_omitRecoveryAttempt` `:1236-1252`) → [[context-edit-overlay]]; abortable `sleep` on `_retryAbortController` (`:3789-3798`); return true → driver `agent.continue()` with `_failedResponse` set (used by virtual router `reason:"retry"`, `agent-session.ts:819-828`) → [[pi--run-settlement|run-settlement]].
  - Delay `baseDelayMs · 2^(attempt−1)` capped at `maxAgentDelayMs` (`retry.ts:128-131`) — 2s/4s/8s; **no jitter** at agent level despite comment "before jitter" (`retry.ts:120`).
  - Counter reset + `auto_retry_end{success:true}` on any non-error assistant `message_end` (`agent-session.ts:1173-1182`; `4f004adef` #1019).
  - Exhausted → `auto_retry_end{success:false, finalError}` (`:1873-1881`); cancelled → `finalError:"Retry cancelled"` (`:3745-3755`).
  - `agent_end.willRetry` precomputed synchronously (`:1197-1211`).
  - Summarization (compaction/branch summary) shares the same `settings.retry` budget via `retryAssistantCall` + `summarization_retry_*` events (`agent-session.ts:3714-3743`; `8e53e0e49`; `packages/coding-agent/src/core/compaction/compaction.ts:614-638`). `retryAssistantCall(produce, policy, signal, callbacks)` (`retry.ts:191-241`): aborted never retried; non-retryable fails fast; caller must handle overflow **before** retry (`:248-250`).
- **Error text is the retry contract** (string classifier): adapters shape messages — Responses/Azure prefix HTTP status (`52e13870a`, #4232); Mistral `finish_reason:"error"` → `"Provider stopped with: error (server error)"` (`mistral-conversations.ts:922-938`; `7fb59f995` #10487); Bedrock keeps `errorMessage` byte-identical with stable prefixes `Internal server error|Model stream error|Throttling error|Service unavailable` and puts metadata in diagnostics "because isRetryableAssistantError matches text" (`bedrock-converse-stream.ts:365-458`, esp. `:435`); `normalizeProviderError`/`formatProviderError` surface status+body (`packages/ai/src/utils/error-body.ts:56-140`) → [[errors-as-stream-events]].
- Per-adapter: Google `retryGoogleRequest` delegates to shared retry and patches `headers = undefined` onto `ApiError` so the guard accepts it — consequence: Google `retry-after`/`RetryInfo` never read (`google-shared.ts:494-515`; `b9d360a2c` #7471). Mistral and pi-messages: no adapter retry (rely on agent level). Codex SSE: own loop `DEFAULT_MAX_RETRIES=0`, base 1000 ms × 2^attempt, terminal-quota veto, `RetryDelayExceededError` (`openai-codex-responses.ts:54-56,123-176,388-461`); WS `previous_response_not_found` / `websocket_connection_limit_reached` → one stateless retry (`:344-352`; `c5dcb2600`, `d0e0b84cb`); transport failure before any event → sticky SSE fallback (`:356-371`; `370fdae6f`).
- OpenAI Decisions classifier: `noRetryStatuses:[504]` — Cloudflare ~5 s edge limit makes 504 deterministic for >600K-token inputs (`ce8972a0e`).

## Constants
| name | value | path:line |
|---|---|---|
| `retry.enabled` | true | `packages/coding-agent/src/core/settings-manager.ts:1011` |
| `retry.maxRetries` | 3 | `settings-manager.ts:1026` |
| `retry.baseDelayMs` | 2000 ms | `settings-manager.ts:1027` |
| `DEFAULT_MAX_AGENT_RETRY_DELAY_MS` (`maxAgentDelayMs`) | 60 000 ms | `packages/ai/src/utils/retry.ts:126` |
| `retry.provider.maxRetryDelayMs` | 60 000 ms | `settings-manager.ts:1061`; `provider-retry.ts:1` |
| provider `maxRetries` default | 0 | `provider-retry.ts:111` |
| provider backoff | `min(0.5·2^i, 8)s`, ≤25% negative jitter | `provider-retry.ts:53-69` |
| Codex SSE `DEFAULT_MAX_RETRIES` / base | 0 / 1000 ms | `openai-codex-responses.ts:54-56` |
| durable `DEFAULT_RETRY_POLICY` | `{3, 2000, 60000}` | `packages/durable/src/harness/agent.ts:19-24` |
| SDK default timeout (passthrough) | 10 min | `packages/ai/src/types.ts:173-177` |

## Evolution
- 2025-12-10 `bb445d24f` agent auto-retry (overloaded/rate limit/5xx), "2s, 4s, 8s" (#157).
- 2025-12-20 `0fc6689df` re-enable SDK's 2 retries for Anthropic — later reversed; same day `c1382818c` "connection error" retryable (#252).
- 2026-01-14 `fb6d464ed` "fetch failed"; 2026-01-22 `9b84857b8` "terminated"; 2026-01-29 `4f004adef` per-response counter reset (#1019).
- 2026-02-01 `030a61d88` `maxDelayMs` cap on server-requested delays (#1123) (later migrated to `retry.provider.maxRetryDelayMs`, `settings-manager.ts:561-582`).
- 2026-03-14 `2501a053e` server_error|internal_error (#2117); 2026-03-17 `8e3bb4ff5` "provider returned error" (#2264); 2026-04-08 `f10cce943` "ended without" (#2892); 2026-04-17 `c15e4d491` connection lost (#3317); 2026-04-23 `2394f3a98` Bedrock http2 (#3594); 2026-05-12 `5ac874c84` Anthropic missing message_stop (#4433).
- 2026-05-18 `52e13870a` status-prefixed Responses/Azure errors (#4232).
- 2026-05-26 `8fb1e877c` **disable hidden provider 429 retries**: SDK `maxRetries` 0 everywhere (Codex 3→0), `isTerminalRateLimitError` for quota/billing (#4991).
- 2026-06-24 `371adcf37` classifier moved from `agent-session.ts` to `packages/ai/src/utils/retry.ts`; explicit "please retry" errors (#6019).
- 2026-07-09 `4285712ba` Bun socket closed (#6431); 2026-07-17 `b0c2a90e5` Responses early EOF (#6727); 2026-07-22 `33e40c3e1` DNS (#6946); `d53b56760` CF 524 (#6239); `e5d18382a` CF 520 (#9627); `57d96d72e` gRPC ResourceExhausted (#6449); `fe10558eb` upstream buffer; `e98f287ee` Azure peak load (#9669).
- 2026-07-21 `243f64be5` aborted retry attempts report `onRetryFinished(false)`.
- 2026-07-23 `7af8533c6` own abortable `retryProviderRequest`; oversized server delays fail fast (#6980/#6911). Commit body lists rejected alternatives: racing SDK request vs signal ("would not let Pi enforce maxRetryDelayMs before the SDK enters its retry sleep"), patching SDK internals, disabling retries.
- 2026-08-03 `b9d360a2c` Google adapters retry transient errors (#7471).
- 2026-09-09 `c37b0e03b` cap agent backoff `maxAgentDelayMs` 60 s (#8826).
- 2026-09-21 `466db0fec` failed attempts omitted from canonical projection.
- 2026-09-30 `2bbfcca43` `Number.isFinite` guard on Retry-After (#9571).
- 2026-10-02 `3874b3e98` "model is at capacity" (#10278); 2026-10-05 `5b6c792b4` HTTP/2 pending stream canceled (#10379, "request was never sent, so safe to retry"); 2026-10-06 `8b5708dbb` server_busy (#10543); 2026-10-07 `7fb59f995` Mistral finish_reason error (#10487); `ce8972a0e` noRetryStatuses 504.

## Evidence commits
`bb445d24f` `0fc6689df` `fb6d464ed` `9b84857b8` `4f004adef` `030a61d88` `2501a053e` `8e3bb4ff5` `f10cce943` `c15e4d491` `2394f3a98` `5ac874c84` `52e13870a` `8fb1e877c` `371adcf37` `4285712ba` `b0c2a90e5` `33e40c3e1` `d53b56760` `e5d18382a` `57d96d72e` `fe10558eb` `e98f287ee` `243f64be5` `7af8533c6` `b9d360a2c` `c37b0e03b` `466db0fec` `2bbfcca43` `3874b3e98` `5b6c792b4` `8b5708dbb` `7fb59f995` `ce8972a0e` `8e53e0e49` `c5dcb2600` `d0e0b84cb` `370fdae6f`

## Quirks
- Unanchored substrings (`"500"`, `"429"`, `"timeout"`, `"terminated"`, `"billing"`) can false-positive (e.g. "1500 tokens" matches 500; "billing" anywhere ⇒ non-retryable) (`retry.ts:20-47,72-73`) (unverified as real-world bug).
- ~52 changelog bullets mention retry classification; classifier grows forever (06-fixes theme table).
- Retry wait shows as `auto_retry_start{delayMs}` so multi-minute outages are visible, but capped at 60 s per attempt.

## Durable variant (packages/durable)
- Retry is a **checkpointed phase**: on retryable error (`!overflow && isRetryableAssistantError && policy.enabled && attempt <= maxRetries`) the generation commits the failed assistant entry and `{phase:"retry", attempt, until}` with an **absolute** deadline `now + retryDelayMs(policy, attempt)` (`packages/durable/src/harness/generation.ts:477-497`), so a crash mid-backoff resumes the wait. Policy read at classification time, not pinned (`:458`). Overflow never retried — "only a compaction can make the next request fit" (`:478`). Recovery resends the same committed messages with the same pinned model/thinking/stream options (`spec.md:3529-3531`).

## Failures
[[retry-wait-race-prompt-returns-early]] · [[retry-counter-accumulates-across-turn]] · [[retry-classifier-regex-sprawl]] · [[retry-backoff-hygiene]] · [[hidden-sdk-retries-double-retry]] · [[deterministic-5xx-retried]] · [[abandoned-attempts-left-in-context]]
