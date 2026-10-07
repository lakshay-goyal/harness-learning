---
type: implementation
harness: codex
concept: persistent-agent-mode
commit: 622e9e3696
files: [codex-rs/prompts/templates/persistent_mode.md:1, codex-rs/core/src/context/world_state/persistent_mode.rs:19, codex-rs/core/src/session/world_state.rs:245, codex-rs/features/src/lib.rs:569, codex-rs/core/src/session/time_reminder.rs:17, codex-rs/core/src/session/turn_context.rs:1059, codex-rs/protocol/src/openai_models.rs:550]
---
[[persistent-agent-mode]] in [[codex]].

## Mechanism
- **Activation**: `persistent_execution_enabled(effort)` ⇔ effective reasoning effort == `ReasoningEffort::Persistent` (`codex-rs/features/src/lib.rs:569-571`; effort value `codex-rs/protocol/src/openai_models.rs:70`) → [[thinking-level-abstraction]].
- **Prompt**: template `codex-rs/prompts/templates/persistent_mode.md` (3,376 bytes, H2 "## Proactivity") rendered as developer fragment `<persistent_mode>…</persistent_mode>`, content kind `persistent_mode.instructions` (`codex-rs/core/src/context/world_state/persistent_mode.rs:19-44`); added as a world-state section each step so toggling emits a diff (`codex-rs/core/src/session/world_state.rs:245-258`; snapshot keeps an object "JSON null would delete the section and lose its disabled state", `persistent_mode.rs:46-49`) → [[world-state-diff-injection]].
- **Catalog override**: `persistent_instructions` — "Missing or null uses the built-in instructions; an empty string disables them" (`codex-rs/protocol/src/openai_models.rs:550-553`) → [[per-model-system-prompt]].
- **Approval channel**: `{{ approval_request_channel }}` → " via functions.send_user_message_async" when the model catalog lists `send_user_message_async` and the session is not a non-root agent, else empty (`codex-rs/core/src/context/world_state/persistent_mode.rs:51-70`; `codex-rs/core/src/session/world_state.rs:245-251`).
- **Defaults**: persistent effort applies `apply_persistent_defaults` to the per-turn config — enables current-time reminders with `sleep_tool: true` unless explicitly configured; "explicit settings and managed policy win" (`codex-rs/core/src/session/time_reminder.rs:17-38`; call `codex-rs/core/src/session/turn_context.rs:1059-1064`) → [[current-time-reminder]], [[wall-clock-tools]].
- **Effort history item**: Persistent normalizes to "disabled" in `configuration_update` items (`codex-rs/core/src/session/reasoning_effort.rs:151`) → [[mid-turn-settings-switch]].
- **Key sentences** (template): "After you've completed the user task and delivered the final answer, if you are sampled again without a new user request, look for useful follow-ups that directly support the completed work… not to infer new authorization."; "Being sampled again or receiving environment-only context is not a new user request and does not itself warrant a message."; "Before starting a follow-up, identify its scope, the outcome you want to establish, the evidence needed, and a stopping condition"; "Bound a follow-up by its purpose, scope, and outcome, not an arbitrary number of checks. A pending, running, inconclusive, or unchanged result is not by itself completion."; "use short, proportionate waits, often 1–3 minutes for active near-term work"; "Persistence does not broaden that scope."; avoid "announcing a 'follow-up task,' declaring 'the follow-up is complete,' narrating internal task bookkeeping".

## Constants
| name | value | path:line |
|---|---|---|
| suggested wait cadence | "often 1–3 minutes" for active near-term work | `codex-rs/prompts/templates/persistent_mode.md` |
| template size | 3,376 bytes | `codex-rs/prompts/templates/persistent_mode.md` |

## Evolution
- 2026-08-27 `f1433fc71f` "Add developer instructions for persistent mode (#41050)".

## Versus pi
- pi has no re-sampling without user input; the closest is an extension follow-up message ([[pi--follow-up-queue]]).
