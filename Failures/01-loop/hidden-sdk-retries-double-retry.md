---
type: failure
concepts: [auto-retry-backoff]
harnesses: [pi, opencode]
---
**Symptom** — Provider SDK default retries (2–3) ran underneath pi's own retry loop: quota/billing 429s were retried pointlessly and burned quota, user-visible errors were delayed, `maxRetries` was unhonored on Codex.

**Root cause** — Two retry layers, the lower invisible; terminal rate limits (`insufficient_quota`, billing) indistinguishable from transient 429 at the SDK level. Policy flip-flopped: 0.25.1 re-enabled SDK retries for fast transient recovery (`0fc6689df` 2025-12-20; same-day `c1382818c` made "connection error" agent-retryable, #252).

**Fix · [[pi]]** — `8fb1e877c` 2026-05-26 "disable hidden provider 429 retries" (#4991): SDK `maxRetries` default 0 everywhere (Codex `DEFAULT_MAX_RETRIES` 3→0, `packages/ai/src/api/openai-codex-responses.ts:54-56`); `isTerminalRateLimitError` / non-retryable pattern for `insufficient_quota|quota exceeded|billing|…` (`packages/ai/src/utils/retry.ts:7-28`). Followed by `7af8533c6` (own abortable adapter retry, opt-in `retry.provider.maxRetries`).

**Fix · [[opencode]]** design change, no incident recorded: `2528d8cb88` 2025-07-03 relied on hidden AI SDK `maxRetries: 10`; `7c7ebb0a9d` 2025-10-22 replaced it with a visible session retry and SDK `maxRetries: 0` (`packages/opencode/src/session/llm.ts:323`). Remaining hidden layers: title side call `retries: 2` (`packages/opencode/src/session/prompt.ts:234`) and Responses WebSocket `streamRetries ?? 5` (`packages/opencode/src/plugin/openai/ws-pool.ts:37`).

**Lesson** — Exactly one visible retry layer owned by the harness; classify quota/billing as terminal.

Related: [[auto-retry-backoff]] · [[retry-backoff-hygiene]] · [[deterministic-5xx-retried]] · [[pi--auto-retry-backoff|pi]] · [[opencode--auto-retry-backoff|opencode]]
