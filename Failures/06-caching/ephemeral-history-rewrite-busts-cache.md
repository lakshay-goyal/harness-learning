---
type: failure
concepts: [cache-stable-prompt-prefix, steering-queue]
harnesses: [opencode]
---
**Symptom** — Sessions where the user typed while the agent worked lost the prompt cache on the following requests: a user message that had already been sent was re-sent with different bytes.

**Root cause** — Messages queued during a run were wrapped, only in the request that first carried them (step > 1), in `<system-reminder>The user sent the following message: … Please address this message and continue with your tasks.</system-reminder>`. On the next request the same message went out unwrapped, so the prefix diverged at that message.

**Fix · [[opencode]]** — `f991fbbde8` 2026-01-02 (#6725) introduced the ephemeral wrapper "to stay on track"; `f092bafe88` 2026-06-19 (#33039) "remove steering wrapper that can bust cache" deleted it. Residual instance of the same pattern at HEAD (legacy): without the experimental plan-mode flag, the plan / build-switch reminders are pushed ephemerally onto whichever user message is currently newest (`packages/opencode/src/session/reminders.ts:23-47`), so when the next user message arrives the earlier one is re-sent without its reminder — a prefix divergence once per user turn rather than per step; under the experimental flag they are persisted as synthetic parts via `sessions.updatePart` (`packages/opencode/src/session/reminders.ts:51-88`) → [[ephemeral-reminder-injection]]. v2 lists "Steering, plan/build-switch, and final-step reminders" as missing and adds only reminders "whose behavior remains part of V2" (`specs/v2/session.md:141`).

**Lesson** — A request-time transform of history must be a pure function of the stored message for its whole lifetime; anything that depends on "is this the first time it is sent" belongs in a new appended message, not in a rewrite.

Related: [[cache-stable-prompt-prefix]] · [[steering-queue]] · [[context-transform-hook]] · [[ephemeral-reminder-injection]] · [[opencode--steering-queue|opencode steering]]
