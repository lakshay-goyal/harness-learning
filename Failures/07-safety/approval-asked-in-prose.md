---
type: failure
concepts: [permission-state-prompt]
harnesses: [codex]
---
**Symptom** — Instead of setting the escalation parameter on the tool call, the model asked the user for permission in chat (or messaged before the call), stalling non-interactive runs; later it also used the structured question tool for permission requests.

**Root cause** — Two channels could express "may I?"; the prompt preferred neither, and models default to conversational asks.

**Fix · [[codex]]** — 2025-11-13 `8dcbd29edd` "ALWAYS proceed to use the `with_escalated_permissions` and `justification` parameters. Within this harness, prefer requesting approval via the tool over asking in natural language."; 2025-12-12 `9429e8b219` "do not message the user before requesting approval for the command." (now `codex-rs/prompts/templates/permissions/approval_policy/on_request.md`); 2026-08-28 `a5c581e247` "Never use the `request_user_input` tool for permission requests or permission-related escalations." (`codex-rs/collaboration-mode-templates/templates/default.md`).

**Lesson** — Give the model exactly one channel for each kind of ask and name the wrong channels explicitly.

Related: [[permission-state-prompt]] · [[sandbox-escalation-retry]] · [[structured-user-question-tool]] · [[codex--permission-state-prompt|codex]]
