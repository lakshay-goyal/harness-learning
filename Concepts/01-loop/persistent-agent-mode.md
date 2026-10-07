---
type: concept
stage: loop
tier: variant
aliases: [persistent-mode-prompt, "ReasoningEffort::Persistent", persistent_instructions, "<persistent_mode>", "## Proactivity", PersistentModeState, persistent_execution_enabled, send_user_message_async]
harnesses: [codex]
---
An "always-on" mode in which the model may be re-sampled after its final answer without a new user request and must decide on bounded, authorized follow-ups (verify, poll, close loops) or stay quiet; guidance lives in a dedicated developer fragment and the mode switches on time awareness and a sleep tool.

## Why
- Agents that keep running between user messages (monitoring, waiting on CI) re-announce, duplicate messages, or invent new work when sampled with no new input.
- Follow-ups need a stopping condition grounded in the task, otherwise "pending / unchanged" results loop forever or get declared done prematurely.
- Waiting needs a clock and an interruptible sleep ([[current-time-reminder]], [[wall-clock-tools]]).

## Design space
- **Activation**: separate mode flag · a reasoning-effort value (✔ codex `ReasoningEffort::Persistent` → `persistent_execution_enabled`).
- **Guidance source**: hard-coded · model-catalog `persistent_instructions` (missing → bundled, empty → disabled) (✔ codex) · developer fragment re-emitted via world state when toggled (✔ codex `<persistent_mode>`).
- **User contact channel**: final answer only · async message tool for approval requests when available (✔ codex `send_user_message_async`, root agent only).
- **Bundled side effects**: force-enable time reminders + `clock.sleep` for the turn unless explicitly configured (✔ codex `apply_persistent_defaults`).
- **Contrast**: goal-driven continuation keeps pushing one objective ([[persistent-goal-continuation]]); persistent mode is open-ended proactivity bounded by prior authorization.

## Implementations
- [[codex--persistent-agent-mode|codex]] — `prompts/templates/persistent_mode.md` (3,376 bytes) as `<persistent_mode>` developer section; activates with Persistent effort; time reminder + sleep tool defaults.

## Failures
none recorded.

## Related
[[persistent-goal-continuation]] · [[current-time-reminder]] · [[wall-clock-tools]] · [[world-state-diff-injection]] · [[message-role-layering]] · [[thinking-level-abstraction]] · [[per-model-system-prompt]]
