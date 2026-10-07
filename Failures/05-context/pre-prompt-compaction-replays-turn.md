---
type: failure
concepts: [auto-compaction, run-settlement]
harnesses: [pi]
---
**Symptom** — When the pre-prompt compaction check fired (on the last assistant message, including aborted ones), it called `agent.continue()` and replayed/continued the old turn before the user's new prompt was sent.

**Root cause** — The pre-prompt check reused the post-run logic, whose return value means "continue the agent".

**Fix · [[pi]]** — `73581ea99` 2026-06-25 "avoid pre-prompt compaction continue": in `prompt()` the check's result is ignored — comment "The user's new prompt is sent below, so do not call agent.continue() here." (`packages/coding-agent/src/core/agent-session.ts:2051-2056`).

**Lesson** — Don't auto-continue when the user is about to send a new message; compaction checks have different follow-ups depending on where they run.

Related: [[auto-compaction]] · [[run-settlement]] · [[queued-messages-stranded-at-run-end]]
