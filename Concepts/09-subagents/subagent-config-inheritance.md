---
type: concept
stage: subagents
tier: variant
aliases: [build_agent_spawn_config, build_agent_shared_config, apply_spawn_agent_runtime_overrides, prepare_agent_spawn_config, default_subagent_model, default_subagent_reasoning_effort, subagent_developer_instructions, MAX_SPAWN_AGENT_MODEL_OVERRIDES, apply_spawn_agent_service_tier]
harnesses: [codex]
---
An explicit rule set for what a child agent copies from the parent's live turn (model, provider, effort, instructions, approval/sandbox policy, environments, exec policy, service tier) versus what its role, spawn arguments or configured defaults override.

## Why
- Cloning stored config instead of the live turn "can send the child agent out with the wrong provider or runtime policy" — a long tail of inheritance bugs ([[subagent-config-not-inherited]]).
- Defaults that differ from the parent invite silent model downgrades ([[subagent-model-downgrade]]).
- Approvals the user persisted for one agent should cover its children, or delegation multiplies prompts.

## Design space
- **Source**: persisted config (bug-prone) · live turn context via one helper (✔ codex `build_agent_shared_config` + `apply_spawn_agent_runtime_overrides`) · explicit model + inherited thinking (pi example).
- **Precedence**: spawn args → configured subagent defaults → parent (✔ codex) · role config ([[agent-profiles]]) as a bounded layer.
- **Validation**: unknown model → error listing a few valid models; effort validated against the model's levels (✔ codex).
- **Shared state**: exec policy (persisted approvals) shared by default (✔ codex); service tier follows the root (✔ codex); environments from the exact requesting step (✔ codex).
- **Instructions**: inherit parent developer instructions · replace with a dedicated subagent instruction block (✔ codex V2 option) · strip parent-specific developer content on forks ([[forked-child-inherits-parent-tool-noise]]).

## Implementations
- [[codex--subagent-config-inheritance|codex]] — `codex-rs/core/src/agent/child_config.rs`: live-turn snapshot (model, provider, effort, instructions, token budget, approvals, cwd, permission profile), overrides, shared exec policy, root service tier.

## Failures
- [[subagent-config-not-inherited]]
- [[subagent-model-downgrade]]
- [[subagent-role-escalates-authority]]

## Related
[[in-process-subagent-threads]] · [[agent-profiles]] · [[task-owned-subagent]] · [[subagent-as-subprocess]] · [[model-resolution]] · [[layered-settings]] · [[command-rule-policy]] · [[builtin-subagents-vs-none]]
