---
type: implementation
harness: codex
concept: cache-warming
commit: 622e9e3696
files: [codex-rs/core/src/client.rs:17-26, codex-rs/core/src/client.rs:2176-2226, codex-rs/core/src/codex_thread.rs:267-284, codex-rs/ext/guardian-v2/src/async_scorer/sampler.rs:154-155, codex-rs/ext/guardian-reviewer/src/pool.rs:148]
---
[[cache-warming]] in [[codex]]. Codex has a variant: a *pre-turn* warmup request, not a TTL keep-alive.

## Mechanism
- **WebSocket prewarm.**
  - It is a v2-only `response.create` with `generate=false`, sent before the first stream request of a turn.
  - It waits for `Completed`, so the real request can reuse the same connection and the warmup's `previous_response_id` (`codex-rs/core/src/client.rs:17-21,2176-2226`).
  - It is skipped if WebSockets are disabled or the connection is already prewarmed.
  - It "is treated as the first websocket connection attempt for a turn. If it fails, normal stream retry/fallback logic handles recovery on the same turn." (`codex-rs/core/src/client.rs:23-26`).
  - On `FallbackToHttp` it switches the session to the HTTPS transport (`codex-rs/core/src/client.rs:2215-2218`).
- **Idle-thread warmup** (`codex-rs/core/src/codex_thread.rs:267-284`):
  - `prewarm()` schedules the empty-input startup warmup (`PrewarmInput::Base`).
  - `prewarm_with_history()` (`PrewarmInput::History`) prepares a response from the existing conversation history and executed-tool metadata with `generate: false`. "The next turn reuses the prepared response only if its prompt still extends this history and its request settings match."
- **Side-model pools** prewarm too:
  - the Guardian v2 async scorer connection pool (`codex-rs/ext/guardian-v2/src/async_scorer/sampler.rs:154-155`);
  - the Guardian reviewer pool (`codex-rs/ext/guardian-reviewer/src/pool.rs:148`).
- **What is warmed.** The connection and server-side response state, i.e. the `previous_response_id` continuation. Warming the provider prompt cache as a side effect of a `generate=false` request with the history prefix is inferred, not stated in code (unverified).
- **Never gates the turn.** `turn/started` is emitted immediately and consuming the prewarm is cancellable (`codex-rs/core/src/tasks/regular.rs:45-80`; `6ea041032b`).
- **No TTL model and no cost gate.** There is no cache-TTL metadata, no refresh timer and no expected-value check. Warmup fires at thread start, at idle-thread preparation, or before the first request of a turn.

## Evolution
- `e416e578bb` 2026-02-06: WebSocket preconnect.
- `6ea041032b` 2026-03-17 (#14838): 15 s startup timeout. Prewarm had blocked `turn/start` for up to 5 min; `turn/started` is now emitted immediately and interrupt is allowed during warmup ([[prewarm-blocks-turn-start]]).
- `7769bccbb2` 2026-09-07: avoid WebSocket waits in Guardian classification.
- `3f4668da20` 2026-09-27 (#48812): history-aware prewarming for idle threads. Telemetry is labeled by input mode, and continuation metrics carry `after_prewarm`.

## Versus pi
- [[pi--cache-warming|pi]] replays the last request with a 1-token output cap shortly before Anthropic's TTL expires, gated on expected value ($0.05 threshold).
- Codex's warmup is latency-oriented (connection plus prepared response) and runs ahead of the turn; it is not a keep-alive against expiry.
- Both treat the warmup as invisible to context.

## Failures
- [[prewarm-blocks-turn-start]]
- [[stream-stall-without-header-timeout]]
