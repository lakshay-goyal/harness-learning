---
type: concept
stage: loop
tier: variant
aliases: [StepSettings, ResolvedStepSettings, next_step_settings, "Op::TurnSettings", apply_turn_settings, StepModelSwitching, reasoning_effort_override, configuration_update item, per-step settings snapshot]
harnesses: [codex]
---
Model, reasoning effort and approval/permission settings can change while a turn runs; the change is captured as an immutable per-step snapshot applied at the next sampling request, never to the in-flight one.

## Why
- Long turns outlive the user's intent: switching model/effort mid-task without waiting for the turn to end.
- In-flight requests and running tool calls must keep the settings they started with; mutating shared config mid-step gives tools and requests inconsistent policy.
- Rewriting the system prompt for a change invalidates the cache prefix; an appended note does not ([[cache-preserving-config-update]]).

## Design space
- **Granularity**: per user turn (common) · per sampling step (✔ codex `StepSettings` snapshot captured per `StepContext`).
- **Targeting**: global setting · addressed to a specific running turn id, rejected if cancelled (✔ codex `Op::TurnSettings{turn_id, update}`).
- **Model notification**: silent · trusted `configuration_update` history item for effort changes (✔ codex, flag-gated) · world-state diff for model/permission sections ([[world-state-diff-injection]]).
- **Validation**: accept anything · validate against proposed permissions (✔ codex `1d64085e67`).
- **Gating**: always on · feature flag (✔ codex `Feature::StepModelSwitching`).

## Implementations
- [[codex--mid-turn-settings-switch|codex]] — `Op::TurnSettings` → `apply_turn_settings` replaces the next-step snapshot; "Replacing the snapshot does not change steps or actions that have already captured it".

## Failures
- [[tool-loadout-stale-within-run]] (same class: per-request recapture)

## Related
[[turn-loop]] · [[turn-lifecycle-hooks]] · [[world-state-diff-injection]] · [[cache-preserving-config-update]] · [[thinking-level-abstraction]] · [[model-resolution]] · [[approval-policy-modes]]
