---
type: failure
concepts: [abort-propagation, unified-provider-api]
harnesses: [pi]
---
**Symptom** — Esc did not interrupt during "Working…" (Gemini CLI stream ignored the signal; escape handler not restored); Codex SSE body reads continued after abort; a mid-stream abort crashed the process via an unhandled undici client `error` event.

**Root cause** — Abort reached the request but not every stream reader / transport object: some readers never observed the signal, and Node `EventEmitter` "error" without a listener is fatal.

**Fix · [[pi]]**
- `a65da1c14` 2026-01-08 — ESC key not interrupting during Working… state.
- `a36a132c7` 2026-05-29 — abort Codex SSE body reads (reader cancelled on abort, `packages/ai/src/api/openai-codex-responses.ts:805-808`).
- `2117b61c6` 2026-06-30 — handle undici mid-stream client errors (no-op listener).
- Contract at HEAD: adapters set `stopReason = signal.aborted ? "aborted" : "error"` and emit partial output (`packages/ai/src/api/anthropic-messages.ts:887-897`); per-provider abort tests "should abort mid-stream", "should handle immediate abort" (`packages/ai/test/abort.test.ts:100-347`).

**Lesson** — Every provider stream reader must observe the signal and cancel its body; test abort per adapter.

Related: [[abort-propagation]] · [[unified-provider-api]] · [[errors-as-stream-events]] · [[pi--abort-propagation|pi]]
