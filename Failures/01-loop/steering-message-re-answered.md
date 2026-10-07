---
type: failure
concepts: [persistent-goal-continuation, auto-compaction]
harnesses: [codex]
---
**Symptom** — Hidden developer-role goal steering messages from earlier turns were "responded to again after a new turn", and fared badly through compaction (`96836e15ed` body).

**Root cause** — Role choice: developer messages persist as standing instructions and are treated differently by compaction than user messages.

**Fix · [[codex]]**
- `96836e15ed` 2026-05-11 "Improve goal continuation based on feedback (#22045)": steering emitted as hidden user-context fragments (`InternalModelContextFragment` via `ContextualUserFragment`, `codex-rs/ext/goal/src/steering.rs:60-66`) — "works better with compaction because user messages are treated differently from developer messages during compaction".

**Lesson** — The role of injected harness text changes how persistently the model obeys it and how compaction treats it; one-shot nudges belong in turn-scoped user context, not standing developer instructions.

Related: [[persistent-goal-continuation]] · [[auto-compaction]] · [[message-role-layering]] · [[xml-prompt-boundaries]] · [[codex--persistent-goal-continuation|codex]]
