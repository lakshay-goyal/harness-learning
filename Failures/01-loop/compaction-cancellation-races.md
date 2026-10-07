---
type: failure
concepts: [abort-propagation, auto-compaction, run-settlement]
harnesses: [pi, codex]
---
**Symptom** — Cancellation could start an auto-compaction anyway, leave stale retry state, or be missed while waiting for summarization auth; RPC `abort` reported success while manual compaction kept running; manual and threshold compaction raced; tree navigation replaced an active compaction's UI; dispose left work running.

**Root cause** — Compaction/branch summary are a second "run" without the main loop's abort plumbing and mutual exclusion: abort classified by error-string matching (`message === "Compaction cancelled" || AbortError`), auth await not abortable, no abort checkpoints, "idle" excluded compaction.

**Fix · [[pi]]**
- `5b31ffd74` 2026-05-28 — dispose aborts agent/compaction/summary/retry/bash (`packages/coding-agent/src/core/agent-session.ts:1392-1413`) (#5029).
- `e56893f4c` 2026-08-03 — prevent auto-compaction race during manual compaction (#7370).
- `bea67d90d` 2026-09-03 — session abort cancels compaction; compaction + branch summary included in idle tracking (#8920).
- `e687434a6` 2026-09-07 — reject tree navigation during compaction (#9179).
- `de2de549b` 2026-09-19 — signal-based classification, abortable summarization auth, `throwIfAborted()` checkpoints (`agent-session.ts:3115-3119,3144,3174`) (#9340, #9777).

**Fix · [[codex]]** (variant: compaction ordering vs turn lifecycle)
- Symptom: auto-compaction could start before `TurnStarted` was emitted, breaking client ordering → `26590d7927` 2026-01-28 "Ensure auto-compaction starts after turn started (#10129)".
- Symptom: context updates recorded before pre-turn compaction got summarized away and not reinjected → `bb0ac5be70` 2026-02-20 "Fix compaction context reinjection and model baselines (#12252)".
- Symptom: pre-turn compaction ran before incoming input was recorded; a failure left the accepted prompt out of history and error events raced prompt hooks → `ee93abb690` 2026-09-10 "Preserve incoming prompts when pre-turn compaction fails (#44487)" (`codex-rs/core/src/compact.rs:250` "Pre-turn failures are reported after preserving the incoming prompt"; `codex-rs/core/src/session/turn.rs:184-224`).
- Structural: manual compaction is a `CompactTask` in the single task slot (`codex-rs/core/src/tasks/compact.rs:63-95`), so it is aborted/replaced through the same lifecycle as turns ([[single-active-task-slot]]); compaction aborts immediately on `SessionBudgetExceeded` (`codex-rs/core/src/compact.rs:335-337`) and its model fallback skips abort/interrupt/budget errors (`codex-rs/core/src/compact_model_fallback.rs:10-32`).

**Lesson** — Classify cancellation by signal state, never by error text; "idle" must include every background LLM operation; side phases need the same abort plumbing, mutual exclusion and an explicit order relative to turn lifecycle events (start → record input → compact → re-inject context → sample).

Related: [[abort-propagation]] · [[auto-compaction]] · [[run-settlement]] · [[side-phase-input-lost]] · [[pi--abort-propagation|pi]] · [[single-active-task-slot]] · [[compaction-drops-pending-prompt]] · [[codex--run-settlement|codex]]
