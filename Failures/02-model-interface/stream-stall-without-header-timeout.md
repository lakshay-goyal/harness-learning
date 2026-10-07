---
type: failure
concepts: [http-transport-hardening]
harnesses: [pi, codex]
---
**Symptom** — The Codex SSE request sat at "Working..." forever with zero events (#4945). The first 10 s fix then produced false timeouts.

**Root cause** — There was no bound on waiting for response headers, and the first bound was a magic constant unrelated to the user's timeout.

**Fix · [[pi]]**
- `7c02a5563` 2026-05-27 — 10 s header timeout.
- `be7d5cf58` 2026-06-12 — relaxed to 20 s after false timeouts.
- `54113731b` 2026-06-28 — use the configured HTTP `timeoutMs` via `AbortSignal.timeout(timeoutMs)` (`packages/ai/src/api/openai-codex-responses.ts:396-413`). The coding-agent passes `timeoutMs` = provider retry setting ?? HTTP idle timeout (0 → 2147483647) (`packages/coding-agent/src/core/sdk.ts:340-366`).
- Related live quirk: Mistral `AbortSignal.timeout(options.timeoutMs ?? 60_000)` spans the **whole** request, including body streaming (`packages/ai/src/api/mistral-conversations.ts:307-308`). Generations over 60 s are cut if no `timeoutMs` is given (unverified whether the coding-agent always sets it).

**Fix · [[pi]]** (WebSocket side) — `493efd422` 2026-05-27 bounded waits for Codex WebSockets (#4979, for #4945). Stall detection needs separate connect / first-byte / inter-event budgets.

**Fix · [[codex]]**
- Symptom: a stalled WebSocket *send* never reached the receive-side idle timeout. Sessions "recover[ed] only after a long quiet period when the server had already logged the websocket as disconnected".
- `35aaa5d9fc` 2026-05-01 (#20751): bound WebSocket sends with the idle timeout (`codex-rs/codex-api/src/endpoint/responses_websocket.rs:693,897-903`).
- Connect-phase hang: a 15 s connect timeout (`6ea041032b` 2026-03-17; [[prewarm-blocks-turn-start]]).
- SSE: every `next()` is wrapped in a 300_000 ms idle timeout, which yields the retryable "idle timeout waiting for SSE" (`codex-rs/codex-api/src/sse/responses.rs:536-569`; `codex-rs/model-provider-info/src/lib.rs:66`).

**Lesson** — Every phase of a request (connect, headers, send, inter-event receive) needs its own deadline. Tie each one to user-configurable timeouts rather than magic constants, and never apply a header timeout to body streaming.

Related: [[http-transport-hardening]] · [[pi--http-transport-hardening|pi]] · [[transport-defaults-kill-connections]] · [[codex--http-transport-hardening|codex]] · [[prewarm-blocks-turn-start]]
