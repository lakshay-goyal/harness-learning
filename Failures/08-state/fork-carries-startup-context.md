---
type: failure
concepts: [session-fork]
harnesses: [codex]
---
**Symptom** — Last-N fork truncation returned the *full* rollout (including the startup-context prefix) whenever the history was shorter than the requested window; separately, first-turn model-switch instructions survived a rollback and were duplicated on cold resume.

**Root cause** — Fork/rollback cut by message count but treated harness-injected context (startup context, `ModelSwitchInstructions`, `PersistentModeState` developer content) as belonging to no turn, so it was neither cut nor re-derived.

**Fix · [[codex]]**
- `61cbf3574e` 2026-05-27 (#24751) — fix last-N truncation when history is shorter than the window.
- `a17da5e6e4` 2026-08-06 (#37260) — rollback strips `ModelSwitchInstructions` / `PersistentModeState` from first-turn developer content (`codex-rs/core/src/context_manager/history.rs:822-846`); rollback that trims a mixed initial-context developer message clears the context baseline so canonical context is re-injected (`codex-rs/core/src/context_manager/history.rs:983-995`).
- Partial (last-N) sub-agent forks later removed altogether (`6221a217e2` 2026-10-06) → [[no-partial-history-fork]].

**Lesson** — Forks and rollbacks must prune (or re-derive) harness-injected context tied to the removed turns, not just the conversational messages.

Related: [[session-fork]] · [[context-projection]] · [[world-state-diff-injection]] · [[codex--session-fork|codex]] · [[forked-child-inherits-parent-tool-noise]]
