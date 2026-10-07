---
type: failure
concepts: [overflow-recovery, auto-compaction]
harnesses: [opencode]
---
**Symptom** — With `compaction.auto: false`, a provider context-overflow error still triggered a compaction run; the user's opt-out only covered the proactive threshold path.

**Root cause** — The opt-out was checked in the threshold predicate only (`packages/opencode/src/session/overflow.ts:28`, `isOverflow` returns false when `compaction.auto === false`). The error path in the processor set `ctx.needsCompaction = true` on any `ContextOverflowError` without consulting config.

**Fix · [[opencode]]** `7e09660c3b` 2026-06-04 (#30749) "respect disabled auto compaction on overflow": legacy `halt` now checks `compaction?.auto === false && !ctx.assistantMessage.summary` and instead marks the assistant message `finish: "error"`, publishes `Session.Event.Error`, sets status idle and returns (`packages/opencode/src/session/processor.ts:620-630`). v2 at HEAD: `compactIfNeeded` gates on `config.auto` (`packages/core/src/session/compaction.ts:232-233`) but `compactAfterOverflow` has no such gate (`packages/core/src/session/compaction.ts:178-184`) and is passed unconditionally as `recoverOverflow` (`packages/core/src/session/runner/llm.ts:379`), so v2 still compacts on overflow errors with auto off (inference from code, unverified at runtime).

**Lesson** — Every compaction entry point (threshold, pre-request, provider error) must consult the same opt-out; centralize the gate rather than repeating it per caller.

Related: [[overflow-recovery]] · [[auto-compaction]] · [[overflow-compaction-cascade]] · [[opencode--overflow-recovery|opencode impl]]
