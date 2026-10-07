---
type: failure
concepts: [abort-propagation, unified-provider-api, process-tree-kill]
harnesses: [pi, codex]
---
**Symptom** — Esc did not interrupt during "Working…" (Gemini CLI stream ignored the signal; escape handler not restored); Codex SSE body reads continued after abort; a mid-stream abort crashed the process via an unhandled undici client `error` event.

**Root cause** — Abort reached the request but not every stream reader / transport object: some readers never observed the signal, and Node `EventEmitter` "error" without a listener is fatal.

**Fix · [[pi]]**
- `a65da1c14` 2026-01-08 — ESC key not interrupting during Working… state.
- `a36a132c7` 2026-05-29 — abort Codex SSE body reads (reader cancelled on abort, `packages/ai/src/api/openai-codex-responses.ts:805-808`).
- `2117b61c6` 2026-06-30 — handle undici mid-stream client errors (no-op listener).
- Contract at HEAD: adapters set `stopReason = signal.aborted ? "aborted" : "error"` and emit partial output (`packages/ai/src/api/anthropic-messages.ts:887-897`); per-provider abort tests "should abort mid-stream", "should handle immediate abort" (`packages/ai/test/abort.test.ts:100-347`).

**Fix · [[codex]]** (variant: tool runtimes ignoring the turn's cancellation)
- Symptoms: js_repl execs kept running after an explicit interrupt (`d9a403a8c0` 2026-03-12 "[js_repl] Hard-stop active js_repl execs on explicit user interrupts (#13329)"); active code-mode cells survived the turn (`509565820f` 2026-08-07 "Interrupt active code-mode cells with their turn (#37483)"); interrupted turns could "drop a pending dynamic tool handler before it publishes its completion" and blocked hooks could stall cancellation (`18e28fe1b9` 2026-10-07 #51556).
- HEAD: provider stream polled `.or_cancel(&preempt).or_cancel(&cancellation_token)` (`codex-rs/core/src/session/turn.rs:2625-2630`); tool dispatch spawned under `AbortOnDropHandle`, readiness/lock/hook waits cancellable, `finishes_on_cancellation` lets dynamic handlers publish their own terminal item, otherwise synthetic "aborted by user" output (`e95abcdf49:codex-rs/core/src/tools/parallel.rs:135-305`, `:363-380`); code-mode interrupt in `handle_task_abort` (`codex-rs/core/src/tasks/mod.rs:942-952`).

**Lesson** — Every stream reader and tool runtime needs an explicit cancellation contract that observes the signal; a hard abort must still produce a paired result; test abort per adapter/runtime.

Related: [[abort-propagation]] · [[unified-provider-api]] · [[errors-as-stream-events]] · [[pi--abort-propagation|pi]] · [[code-mode]] · [[codex--abort-propagation|codex]]
