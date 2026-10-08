---
type: implementation
harness: codex
concept: mid-turn-settings-switch
commit: 622e9e3696
files: [codex-rs/core/src/session/step_settings.rs:2, codex-rs/core/src/session/handlers.rs:640, codex-rs/core/src/session/step_activation.rs:229, codex-rs/core/src/session/turn.rs:460, codex-rs/core/src/session/turn.rs:517, codex-rs/core/src/session/reasoning_effort.rs:140]
---
[[mid-turn-settings-switch]] in [[codex]].

## Mechanism
- **Steps**: "A turn may contain several steps, each using its own captured settings" (`codex-rs/core/src/session/step_settings.rs:2-16`); "Replacing the snapshot does not change steps or actions that have already captured it" (`:17-21`).
- **Op**: `Op::TurnSettings{turn_id, update}` → `apply_turn_settings`, gated by `Feature::StepModelSwitching`, targets only the named non-cancelled task (`codex-rs/core/src/session/handlers.rs:640-648`, `codex-rs/core/src/session/step_activation.rs:229-262`); reply via oneshot is the routing decision ([[agent-event-stream]]).
- **Capture point**: a fresh `StepContext` (tools, MCP binding, settings, AGENTS.md) is captured before each sampling request (`codex-rs/core/src/session/turn.rs:460-495`) → [[turn-loop]]. Per-turn config derived from step settings (service tier, persistent-mode defaults) (`codex-rs/core/src/session/turn_context.rs:1055-1066`).
- **Model notification**: effort changes recorded as a trusted `configuration_update` item behind `reasoning_effort_override` (`codex-rs/core/src/session/turn.rs:517-518`); `Persistent` normalizes to "disabled", unknown custom values excluded "so injected items stay bounded to known backend modes" (`codex-rs/core/src/session/reasoning_effort.rs:140-157`). Model / permission changes reach the model as world-state section diffs ([[world-state-diff-injection]]) — appended, never prefix rewrites ([[cache-preserving-config-update]]).
- **Validation**: step settings validated against proposed permissions (`1d64085e67`).
- **Thread vs turn settings**: `Op::ThreadSettings` persistent settings "apply on Started and Steered" (`codex-rs/core/src/session/turn_input.rs:8`).

## Evolution
- 2026-08-25 `68301fa45f` "Snapshot resolved settings for each model step (#40651)"; same day `1d64085e67` "Validate step settings against proposed permissions (#40647)".
- 2026-09-05 `56a8470aa0` "Record reasoning effort changes in conversation history behind a flag (#43110)".

## Versus pi
- pi recomputes model/thinking/tools per request through `prepareNextTurn` hooks ([[pi--turn-lifecycle-hooks]]) without a typed per-step snapshot or turn-addressed op; codex makes the snapshot immutable per step and addresses updates to a turn id.
