---
type: failure
concepts: [plan-mode, ephemeral-reminder-injection]
harnesses: [opencode]
---
**Symptom** — (Inferred from the fixes.) After the user switched from plan to build, earlier plan-mode reminders were still in the history, so the model kept acting read-only; the first lifting message ("You are now in build mode and are permitted to make edits") did not mention the shell.

**Root cause** — Restrictive reminders are injected into user messages and stay in the transcript; nothing told the model the restriction had ended, or only partly.

**Fix · [[opencode]]** — `9231043eb4` 2025-08-21 replaced the one-liner with `build-switch.txt` "Your operational mode has changed from plan to build. You are no longer in read-only mode…"; `bdc0f7c86d` 2025-09-09 wrapped it in `<system-reminder>` and extended it ("make file changes, run shell commands"). Injected whenever an earlier assistant message came from `plan` and the agent is now `build` (`packages/opencode/src/session/reminders.ts:37-47`).

**Lesson** — Every restrictive reminder needs a matching, explicit lifting reminder covering the same capabilities.

Related: [[plan-mode]] · [[ephemeral-reminder-injection]] · [[opencode--plan-mode|opencode]]
