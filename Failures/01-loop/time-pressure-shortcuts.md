---
type: failure
concepts: [persistent-goal-continuation, guideline-softening]
harnesses: [codex]
---
**Symptom** — Showing elapsed time in the goal continuation prompt made the model shortcut work: "I saw evidence that this was causing the model to shortcut work as it became nervous about time" (`96836e15ed` body).

**Fix · [[codex]]**
- `96836e15ed` 2026-05-11 removed "Time spent pursuing goal: {{ time_used_seconds }} seconds" from `continuation.md`; kept in `budget_limit.md` (`codex-rs/ext/goal/templates/goals/budget_limit.md:10`) where wrap-up is the desired behavior.

**Lesson** — Every number shown to the model is an implicit instruction; omit pressure signals unless you want the behavior they induce.

Related: [[persistent-goal-continuation]] · [[guideline-softening]] · [[current-time-reminder]] · [[codex--persistent-goal-continuation|codex]]
