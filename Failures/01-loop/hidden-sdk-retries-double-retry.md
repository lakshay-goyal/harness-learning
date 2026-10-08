---
type: failure
concepts: [auto-retry-backoff]
harnesses: [pi, opencode, codex]
---
**Symptom** — Provider SDK default retries (2–3) ran underneath pi's own retry loop: quota/billing 429s were retried pointlessly and burned quota, user-visible errors were delayed, `maxRetries` was unhonored on Codex.

**Root cause** — Two retry layers, the lower invisible; terminal rate limits (`insufficient_quota`, billing) indistinguishable from transient 429 at the SDK level. Policy flip-flopped: 0.25.1 re-enabled SDK retries for fast transient recovery (`0fc6689df` 2025-12-20; same-day `c1382818c` made "connection error" agent-retryable, #252).

**Fix · [[pi]]** — `8fb1e877c` 2026-05-26 "disable hidden provider 429 retries" (#4991): SDK `maxRetries` default 0 everywhere (Codex `DEFAULT_MAX_RETRIES` 3→0, `packages/ai/src/api/openai-codex-responses.ts:54-56`); `isTerminalRateLimitError` / non-retryable pattern for `insufficient_quota|quota exceeded|billing|…` (`packages/ai/src/utils/retry.ts:7-28`). Followed by `7af8533c6` (own abortable adapter retry, opt-in `retry.provider.maxRetries`).

**Fix · [[codex]]** (variant: non-retryable errors retried across two retry layers)
- Symptom: `insufficient_quota` retried repeatedly in the agent loop (`0c647bc566` 2025-11-06); `invalid_prompt` `response.failed` events retried "despite the prompt being marked as disallowed", hiding the real message (`93a5e0fe1c` 2026-01-16); throttling vs quota misclassified (`31ffe2bc9a` 2026-09-15: `slow_down` retryable, credit/spend-limit codes terminal); "bio policy" errors made non-retryable (`fa8cf44985` 2026-09-17). TS-era inverse: 5xx with `status: undefined` not retried (`e307d007aa` 2025-05-11).
- Layer split: HTTP layer retries only 5xx/transport (`retry_429: false`, `codex-rs/model-provider-info/src/lib.rs:454-460`; `4502b1b263` 2025-11-25), 429 handled once at the stream layer with classification (`codex-rs/protocol/src/error.rs:397-446`). Error-code tables: `codex-rs/codex-api/src/api_bridge.rs:149-167` (`INVALID_PROMPT_ERROR_CODE`), `:221` (`insufficient_quota`); `codex-rs/codex-api/src/sse/responses_error.rs:47-99`.
**Fix · [[opencode]]** design change, no incident recorded: `2528d8cb88` 2025-07-03 relied on hidden AI SDK `maxRetries: 10`; `7c7ebb0a9d` 2025-10-22 replaced it with a visible session retry and SDK `maxRetries: 0` (`packages/opencode/src/session/llm.ts:323`). Remaining hidden layers: title side call `retries: 2` (`packages/opencode/src/session/prompt.ts:234`) and Responses WebSocket `streamRetries ?? 5` (`packages/opencode/src/plugin/openai/ws-pool.ts:37`).

**Lesson** — Own retries in the harness; if two layers exist, give them disjoint error classes, and classify quota/billing/policy/invalid-input as terminal.
Related: [[auto-retry-backoff]] · [[retry-backoff-hygiene]] · [[deterministic-5xx-retried]] · [[pi--auto-retry-backoff|pi]] · [[opencode--auto-retry-backoff|opencode]]

Related: [[auto-retry-backoff]] · [[retry-backoff-hygiene]] · [[deterministic-5xx-retried]] · [[pi--auto-retry-backoff|pi]] · [[codex--auto-retry-backoff|codex]]
