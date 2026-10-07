---
type: implementation
harness: codex
concept: subagent-config-inheritance
commit: 622e9e3696
files: [codex-rs/core/src/agent/child_config.rs:20, codex-rs/core/src/agent/control/spawn.rs:726]
---
[[subagent-config-inheritance]] in [[codex]].

## Mechanism
- **Live-turn source**: child config starts from the parent's *live turn*, not stored config: model slug, provider, reasoning effort/summary, developer instructions, base instructions + provenance, token budget, approval policy, approvals reviewer, cwd, permission profile snapshot (`codex-rs/core/src/agent/child_config.rs` `build_agent_spawn_config` / `build_agent_shared_config` / `apply_spawn_agent_runtime_overrides`). Doc: "skipping this helper and cloning stale config state directly can send the child agent out with the wrong provider or runtime policy".
- **Precedence**: requested `model` / `reasoning_effort` args → `agents.default_subagent_model` / `default_subagent_reasoning_effort` → parent's. Unknown model → error listing up to `MAX_SPAWN_AGENT_MODEL_OVERRIDES` = 5 picker-visible models (`codex-rs/core/src/agent/child_config.rs:20`, `apply_requested_spawn_agent_model_overrides`, `find_spawn_agent_model_name`); effort validated against the model's supported levels.
- **Shared exec policy**: persisted command approvals shared with children by default ("If a subagent requests approval, and the user persists that approval to the execpolicy, it should (by default) propagate", `84f4e7b39d`) → [[command-rule-policy]].
- **Environments** inherited from the exact step that requested the child (`codex-rs/core/src/agent/control/spawn.rs:726-736`; `3749d1eff7`).
- **Service tier** follows the root; dropped if the child model doesn't support it (`apply_spawn_agent_service_tier`; `dc2ccc6843`).
- **Instructions**: V2 may replace inherited developer instructions with `multi_agent_v2.subagent_developer_instructions` (`49025589b0`); forks strip parent-specific developer content ([[codex--in-process-subagent-threads|in-process-subagent-threads]]).
- **Role layer** applied on top, bounded ([[agent-profiles]]). Approvals of workers can be routed to the automatic reviewer, which sees root-user evidence only ([[delegated-authorization-provenance]]).
- **Workspace**: `config.cwd = turn_cwd` — no per-child isolation ([[no-subagent-workspace-isolation]]).

## Constants
| name | value | path:line |
|---|---|---|
| unknown-model suggestion list | 5 | `codex-rs/core/src/agent/child_config.rs:20` |

## Evolution
- 2026-03-13 `7f571396c8` sync split sandbox policies to spawned subagents.
- 2026-03-16 `33acc1e65f` role honoured under profiles.
- 2026-03-17 `84f4e7b39d` "fix(subagents) share execpolicy by default".
- 2026-04-13 `776246c3f5` forked children keep parent model config.
- 2026-04-20 `54bd07d28c` prefer inherited spawn model (description side).
- 2026-07-16 `21c37fb374` configured subagent model defaults honoured.
- 2026-07-28 `49025589b0` `subagent_developer_instructions`.
- 2026-08-04 `d1f14e31a7` reloaded V2 agents keep model provider.
- 2026-08-28 `dc2ccc6843` "Make subagents follow the root service tier".
- 2026-09-28 `3749d1eff7` "Preserve pending environments when spawning subagents".

## Versus pi
- [[pi--subagent-as-subprocess]]: explicit agent `model` wins, else inherit dispatcher model + thinking; tools come from agent frontmatter, never inherited. Codex inherits the whole runtime policy (incl. approvals, sandbox, environments, exec policy) through one helper.
