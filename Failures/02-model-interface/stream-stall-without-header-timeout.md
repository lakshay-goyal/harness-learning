---
type: failure
concepts: [http-transport-hardening]
harnesses: [pi, opencode]
---
**Symptom** — The Codex SSE request sat at "Working..." forever with zero events (#4945). The first 10 s fix then produced false timeouts.

**Root cause** — There was no bound on waiting for response headers, and the first bound was a magic constant unrelated to the user's timeout.

**Fix · [[pi]]**
- `7c02a5563` 2026-05-27 — 10 s header timeout.
- `be7d5cf58` 2026-06-12 — relaxed to 20 s after false timeouts.
- `54113731b` 2026-06-28 — use the configured HTTP `timeoutMs` via `AbortSignal.timeout(timeoutMs)` (`packages/ai/src/api/openai-codex-responses.ts:396-413`). The coding-agent passes `timeoutMs` = provider retry setting ?? HTTP idle timeout (0 → 2147483647) (`packages/coding-agent/src/core/sdk.ts:340-366`).
- Related live quirk: Mistral `AbortSignal.timeout(options.timeoutMs ?? 60_000)` spans the **whole** request, including body streaming (`packages/ai/src/api/mistral-conversations.ts:307-308`). Generations over 60 s are cut if no `timeoutMs` is given (unverified whether the coding-agent always sets it).

**Fix · [[pi]]** (WebSocket side) — `493efd422` 2026-05-27 bounded waits for Codex WebSockets (#4979, for #4945). Stall detection needs separate connect / first-byte / inter-event budgets.

**Fix · [[opencode]]** five reversals. SSE idle timeout: `69ddc91c35` 2026-03-10 added (2 min) → `8c53b2b470` 2026-03-14 raised to 5 min (long reasoning pauses tripped it) → `d69962b0f7` 2026-03-19 **off by default** → `c7e1fc5e42` 2026-05-29 stall made retryable → `4eb29a64f0` 2026-09-02 5-min default again, `false` opts out. Header timeout: `f965db9e13` 2026-05-26 10 s, OpenAI only → `67caf894e0` 2026-07-19 300 s → `b04697366f` 2026-09-02 300 s for all providers (`packages/opencode/src/provider/provider.ts:35,94-127`). v2 deliberately has none: "Do not impose a universal provider-stream inactivity or absolute timeout" (`specs/v2/schema-changelog.md:557`).

**Lesson** — Bound header waits, but tie the bound to the user-configurable timeout rather than a magic constant, and never apply a header timeout to body streaming.

Related: [[http-transport-hardening]] · [[pi--http-transport-hardening|pi]] · [[transport-defaults-kill-connections]] · [[opencode--http-transport-hardening|opencode]]
