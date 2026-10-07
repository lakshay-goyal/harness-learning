---
type: failure
concepts: [auto-retry-backoff, abort-propagation, steering-queue]
harnesses: [pi, codex]
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

**Fix · [[codex]]**
- Symptom: 10 stream retries with factor 1.3 hammered the backend → `548466df09` 2025-08-07 "[client] Tune retries and backoff (#1956)": 10→5 retries ("10 is a bit excessive"), factor 1.3→2.0; `d32e4f25cf` 2025-08-25 user retry config capped at 100 (`codex-rs/model-provider-info/src/lib.rs:72-75`).
- Symptom: with instant interrupt, new user input waited for an unfinished response or "stream retry backoff" (`f92655d07f` 2026-09-25 #48141) → retry sleep wrapped `.or_cancel(&preempt).or_cancel(&cancellation_token)` (`codex-rs/core/src/session/turn.rs:1705-1729`).
- Backoff delay itself is uncapped (bounded by retry count) (`codex-rs/async-utils/src/backoff.rs:7-17`).

**Lesson** — Every wait in the loop (incl. backoff sleeps) must be abortable by both abort and steer, and bounded; validate every server-supplied number; if you can't cancel the vendor's backoff, re-implement it.

Related: [[auto-retry-backoff]] · [[abort-propagation]] · [[hidden-sdk-retries-double-retry]] · [[pi--auto-retry-backoff|pi]] · [[steering-queue]] · [[server-retry-advice-ignored]] · [[codex--auto-retry-backoff|codex]]
