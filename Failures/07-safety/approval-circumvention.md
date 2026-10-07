---
type: failure
concepts: [permission-state-prompt, tool-call-gate]
harnesses: [codex]
---
**Symptom** — After a denial or sandbox failure the model pursued the same goal through another command or tool (rewrite the command, use a different tool, read a denied file another way).

**Root cause** — Denial text did not say whether a denial was final or escalatable, and nothing told the model that alternate routes count as circumvention.

**Fix · [[codex]]** — 2025-08-05 `d31e149cb1` "When approval is denied or a command fails due to a permission error, do not retry the exact command in a different way." (dropped 2025-08-07 `81b148bda2`); 2026-01-28 `996e09ca24` "don't try and circumvent approvals by using other tools" → 2026-02-03 `968c029471` "Be judicious with escalating, but if completing the user's request requires it, you should do so - don't try and circumvent approvals by using other tools." (`codex-rs/prompts/templates/permissions/approval_policy/on_request.md`); denied reads: "Do not request escalation or additional permissions to read them; these denials are policy restrictions." (`codex-rs/prompts/src/permissions_instructions.rs:382-396`); Guardian deny feedback "must not attempt to achieve the same outcome via workaround, indirect execution, or policy circumvention" (`codex-rs/prompts/src/model_messages/guardian.rs:10-16`) + per-turn denial circuit breaker (`codex-rs/ext/guardian-reviewer/src/circuit_breaker.rs:3-7`).

**Lesson** — Denial text must say whether the denial is escalatable and that workarounds count as circumvention; back it with a hard stop after repeated denials.

Related: [[permission-state-prompt]] · [[tool-call-gate]] · [[llm-approval-reviewer]] · [[codex--permission-state-prompt|codex]]
