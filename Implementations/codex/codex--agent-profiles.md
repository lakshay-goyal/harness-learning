---
type: implementation
harness: codex
concept: agent-profiles
commit: 622e9e3696
files: [codex-rs/config/src/config_toml.rs:776, codex-rs/core/src/agent/role.rs:69, codex-rs/core/src/agent/role.rs:342, codex-rs/core/assets/agent/agent_names.txt, codex-rs/core/src/agent/registry.rs:54, codex-rs/core/src/agent/control/spawn.rs:41]
---
[[agent-profiles]] in [[codex]].

## Mechanism
- **Declaration**: `[agents.researcher] description=…, config_file=…, nickname_candidates=[…]` (`codex-rs/config/src/config_toml.rs:776-800`); loading extracted to crate `codex-rs/agent-roles/src/{agent_role_config.rs,discovery.rs,loader.rs}` (`fb9311db5c`).
- **Built-ins** (`codex-rs/core/src/agent/role.rs:342-416`): `default`; `explorer` (config file `codex-rs/core/assets/agent/builtins/explorer.toml` is 0 bytes; description tells the parent to spawn explorers in parallel and trust their results); `worker` (description demands explicit file ownership and "Always tell workers they are **not alone in the codebase**, and they should not revert the edits made by others", `:381`); `awaiter` commented out "Awaiter is temp removed" (`:386`; `codex-rs/core/assets/agent/builtins/awaiter.toml` still sets `model_reasoning_effort="low"`, `background_terminal_max_timeout=3600000`) → [[no-awaiter-role]].
- **Bounded override** (`codex-rs/core/src/agent/role.rs:69-127`): a role layer may set only developer_instructions, model, reasoning effort/summary, verbosity, personality, service tier; feature keys may only be turned **off** and only for ShellTool, Apps, Plugins, MemoryTool, RequestPermissionsTool; skills may only be disabled. Symlinked user role files rejected → [[subagent-role-escalates-authority]].
- **Exposure**: `agent_type` parameter hidden from the spawn schema unless roles are configured (`expose_agent_type: !turn_context.config.agent_roles.is_empty()`, `codex-rs/core/src/tools/spec_plan.rs`; `9ff47868eb`). Spawn description: "Agent-role guidance … never authorizes spawning by itself" (`367a8a2210`) → [[over-eager-delegation]].
- **Forks**: V1 forbids `agent_type` on full-history forks; V2 allows (`82b17bc724`).
- **Nicknames**: pool `codex-rs/core/assets/agent/agent_names.txt` (101 lines: Euclid, Archimedes, Ptolemy, Hypatia, Avicenna, …) embedded via `include_str!` (`codex-rs/core/src/agent/control/spawn.rs:41`, `:66-86`); role `nickname_candidates` override it. Random pick among unused names; when exhausted the used set is cleared, reset count++ (metric `codex.multi_agent.nickname_pool_reset`, `codex-rs/core/src/agent/registry.rs:282`) and names become "Plato the 2nd", "… the 3rd", "… the 11th" (`codex-rs/core/src/agent/registry.rs:54-71`, `:259-297`). Nickname hidden from the spawn result by default (`668703c23f`).

## Constants
| name | value | path:line |
|---|---|---|
| nickname pool | 101 names | `codex-rs/core/assets/agent/agent_names.txt` |
| default role name | `"default"` (`DEFAULT_ROLE_NAME`) | `codex-rs/core/src/agent/role.rs:342-416` |
| awaiter (disabled) preset | effort low, background terminal timeout 3 600 000 ms | `codex-rs/core/assets/agent/builtins/awaiter.toml` |

## Evolution
- 2026-01-15 `05b960671d` "feat: add agent roles to collab tools (#9275)" (orchestrator role prompt).
- 2026-02-16 `e47045c806` "feat: add customizable roles for multi-agents (#11917)".
- 2026-02-20 `0f9eed3a6f` "feat: add nick name to sub-agents".
- 2026-02-25..27 `81ce645733`, `c53c08f8f9` "calm down awaiter" (description tuning), then `fe439afb81` "chore: tmp remove awaiter".
- 2026-03-16 `33acc1e65f` role honoured under profiles.
- 2026-07-16 `9ff47868eb` hide `agent_type` unless roles configured.
- 2026-08-06 `82b17bc724` "Allow agent roles on full-history forks (#37252)".
- 2026-08-18 `1a6e07a4fe` "Restrict agent roles to bounded configuration overrides" — "Agent roles should customize a child agent without expanding the authority or changing the provider configuration inherited from its parent session".
- 2026-08-24 `fb9311db5c` `codex-agent-roles` crate.

## Versus pi
- [[pi--subagent-as-subprocess]]: markdown agents with frontmatter (tools allowlist, model) appended to the system prompt; repo agents gated by confirmation. Codex roles are TOML layers restricted to narrowing overrides; tools are not role-selectable except turning a few features off.
