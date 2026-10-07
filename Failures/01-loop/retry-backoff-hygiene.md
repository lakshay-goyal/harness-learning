---
type: failure
concepts: [auto-retry-backoff, abort-propagation]
harnesses: [pi, opencode]
---
**Symptom** — Esc did not interrupt provider retry sleeps (#6911/#6980) or "Working…" after auto-retry (#568); long server-requested waits (Gemini CLI) silently stalled the agent (#1123); an unparseable `Retry-After` HTTP-date fired retries immediately (#9571); agent backoff grew unbounded during outages (#8826); retries after abort were reported as success.

**Root cause** — SDK built-in retry timers ignore the request `AbortSignal`; server delays trusted without bounds; `Date.parse` → NaN → `setTimeout(NaN)` = 0; exponential delay uncapped; abort during backoff not normalized.

**Fix · [[pi]]**
- 0.39.0 Esc works during "Working..." after auto-retry (#568).
- `030a61d88` 2026-02-01 `maxDelayMs` cap on server-requested delays: fail fast with "Server requested Ns retry delay" so the visible agent retry handles it (now `retry.provider.maxRetryDelayMs`, `packages/ai/src/utils/provider-retry.ts:1,39-51`) (#1123).
- `243f64be5` 2026-07-21 aborted retry attempts report `onRetryFinished(false)`; abort during backoff → `aborted` message (`packages/ai/src/utils/retry.ts:204-208,229-236`).
- `7af8533c6` 2026-07-23 own abortable `retryProviderRequest`, SDK `maxRetries:0` (`provider-retry.ts:99-128`; `anthropic-messages.ts:554-556`, also azure/codex/completions/responses) (#6980).
- `c37b0e03b` 2026-09-09 cap `retry.maxAgentDelayMs` 60 s (`retry.ts:126-131`) (#8826).
- `2bbfcca43` 2026-09-30 `Number.isFinite` guards → exponential fallback (`provider-retry.ts:55-65`) (#9571).
- Agent-level sleep abortable via `_retryAbortController` (`packages/coding-agent/src/core/agent-session.ts:3789-3798`).

**Fix · [[opencode]]** `9a1dc1ffe4` 2025-12-31: a huge `retry-after` exceeded the int32 `setTimeout` limit, so Node fired it immediately (`TimeoutOverflowWarning`) → hot retry loop; delay clamped to 2 147 483 647 ms. `c78986831c` 2026-08-11: the session retry had **no attempt cap and no jitter** for ~10 months (since `7c7ebb0a9d` 2025-10-22); now `RETRY_MAX_RETRIES = 5` with 0–25 % positive jitter (`packages/opencode/src/session/retry.ts:26-31,80-83,193`). v2 transport retry: 2 attempts, ±20 % jitter, `retry-after` clamped to 10 s (`packages/llm/src/route/executor.ts:36-38,344-351`).

**Lesson** — Every wait in the loop must be abortable and bounded; validate every server-supplied number; if you can't cancel the vendor's backoff, re-implement it.

Related: [[auto-retry-backoff]] · [[abort-propagation]] · [[hidden-sdk-retries-double-retry]] · [[pi--auto-retry-backoff|pi]] · [[opencode--auto-retry-backoff|opencode]]
