---
type: failure
concepts: [review-subagent, out-of-band-message-deferral]
harnesses: [codex]
---
**Symptom** — The user ran `/review`, saw findings in the UI, then asked the main agent to fix "comment 2" — but the main thread's history had no record of the review.

**Root cause** — The side task's output was rendered to the UI only; the parent conversation never received it.

**Fix · [[codex]]**
- `72733e34c4` 2025-09-16 "Add dev message upon review out (#3758)" — inject a `<user_action>` record with the full findings (or an interrupted notice) into parent history. HEAD: `render_review_exit_success` / `render_review_exit_interrupted` (`codex-rs/core/src/tasks/review.rs:227-231`) → `codex-rs/prompts/templates/review/exit_success.xml`, `exit_interrupted.xml`.

**Lesson** — Any UI-visible side task must leave a model-visible trace in the main transcript.

Related: [[review-subagent]] · [[out-of-band-message-deferral]] · [[interrupted-turn-invisible-to-model]] · [[subagent-result-mailbox]] · [[codex--review-subagent|codex]]
