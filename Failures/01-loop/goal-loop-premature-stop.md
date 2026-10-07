---
type: failure
concepts: [persistent-goal-continuation]
harnesses: [codex]
---
**Symptom** — Goals stopped short "even when the agent could still keep making progress": the continuation prompt invited "If the goal has not been achieved and cannot continue productively, explain the blocker … and wait for new input", and the runtime suppressed continuation after a turn with no tool calls (`3d1d164aee` body).

**Root cause** — A heuristic done-signal ("no registry tool calls" ≡ "should stop") plus a prompt-level escape hatch.

**Fix · [[codex]]**
- `3d1d164aee` 2026-05-01 "Remove no-tool goal continuation suppression (#20523)" — removed the sentence and the heuristic.
- `96836e15ed` 2026-05-11 "Improve goal continuation based on feedback (#22045)" — "Keep the full objective intact… leave the goal active"; explicit `update_goal` complete/blocked audits (`codex-rs/ext/goal/templates/goals/continuation.md:9-56`).

**Lesson** — "No tool calls" is not a reliable done-signal; give the model an explicit completion tool + audit and pair it with harness breakers.

Related: [[persistent-goal-continuation]] · [[goal-continuation-runaway]] · [[premature-turn-end]] · [[codex--persistent-goal-continuation|codex]]
