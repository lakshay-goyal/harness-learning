---
type: failure
concepts: [abort-propagation, guideline-softening, partial-message-persistence, transcript-replay-repair]
harnesses: [codex]
---
**Symptom** — After Esc/Ctrl-C, Codex emitted `TurnAborted` to clients but wrote nothing model-visible to history; on the next turn the model "can't tell the previous work was aborted and may resume/repeat earlier actions (including duplicated side effects like re-opening PRs)" (`b236f1c95d` body, issue #9042).

**Root cause** — Interruption was treated as a UI event, not as conversation state.

**Fix · [[codex]]**
- `b236f1c95d` 2026-01-20 "fix: prevent repeating interrupted turns (#9043)" — persist a hidden `<turn_aborted>` message on interrupt. Now recorded and flushed by `handle_task_abort` before `TurnAborted` (`codex-rs/core/src/tasks/mod.rs:972-990`); fragment content kind `generic.turn_aborted` (`codex-rs/core/src/context/turn_aborted.rs:22`), wrapper `:34`, exported `codex-rs/core/src/context/mod.rs:132`.
- Wording softened twice: `09251387e0` 2026-01-26 dropped "Do not continue or repeat work from that turn unless the user explicitly asks"; `2b8d29ac0d` 2026-03-31 dropped "verify current state before retrying". Current text: "The user interrupted the previous turn on purpose. Any running unified exec processes may still be running in the background. If any tools/commands were aborted, they may have partially executed." (`codex-rs/core/src/context/turn_aborted.rs:10`) → [[guideline-softening]].
- `120aa07d81` 2026-04-24 MultiAgentV2 markers made non-user-authored so a system interruption did not look like the user spoke; `28742866c7` 2026-04-24 `agents.interrupt_message` (marker role contextual-user / developer / disabled, `codex-rs/core/src/tasks/mod.rs:102-118`).
- `70a0b1eef8` 2026-07-15 keeps the output-free interrupted prompt in the transcript.

**Lesson** — An abort must leave a durable, model-visible boundary in history, phrased as neutral state rather than prohibitions.

Related: [[abort-propagation]] · [[partial-message-persistence]] · [[transcript-replay-repair]] · [[guideline-softening]] · [[turn-items-lost-on-abort]] · [[interrupt-kills-background-processes]] · [[codex--abort-propagation|codex]]
