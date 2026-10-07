---
type: concept
stage: subagents
tier: candidate
aliases: [subagent-naming, AgentRoleConfig, AgentRoleToml, "[agents.<name>]", agent_type, explorer role, worker role, awaiter role, codex-agent-roles, agent_names.txt, agent_nickname, nickname_candidates, nickname pool]
harnesses: [codex]
---
Named child presets (description + bounded config layer + nickname pool) that the parent model selects via an `agent_type` argument; roles may only narrow authority, never widen it. Children also get human-readable nicknames from a bundled pool.

## Why
- Delegation quality depends on telling the parent *when* each preset fits (explorer vs worker) and the child *how* to behave (file ownership, shared workspace).
- A role file that is a full config layer can change permissions, providers, endpoints, MCP servers — a privilege escalation path via delegation ([[subagent-role-escalates-authority]]).
- Role descriptions read as invitations to spawn ([[over-eager-delegation]]).
- Users need to tell parallel children apart in the UI.

## Design space
- **Definition**: markdown + frontmatter agents (pi [[subagent-as-subprocess]]) · TOML `[agents.<name>]` with description, `config_file`, `nickname_candidates` + built-ins (✔ codex).
- **Override scope**: anything · bounded allowlist (instructions, model, effort/summary, verbosity, personality, service tier; features only *off* for a few; skills only disabled) (✔ codex).
- **Exposure**: always in schema · `agent_type` hidden unless roles configured (✔ codex).
- **Built-ins**: none (pi core) · `default`, `explorer` (empty config, "spawn explorers in parallel and trust their results"), `worker` (explicit file ownership, "not alone in the codebase"); `awaiter` disabled (✔ codex).
- **Fork interplay**: forbid role on full-history forks (codex V1) · allow (codex V2).
- **Naming**: ids · random unique nicknames from a 101-name pool with ordinal suffixes after exhaustion (✔ codex).

## Implementations
- [[codex--agent-roles|codex]] — `codex-rs/core/src/agent/role.rs` + `codex-rs/agent-roles`; bounded overrides, symlinked user role files rejected; nickname pool `agent_names.txt`.

## Failures
- [[subagent-role-escalates-authority]]
- [[over-eager-delegation]]

## Related
[[subagent-config-inheritance]] · [[in-process-subagent-threads]] · [[task-owned-subagent]] · [[subagent-as-subprocess]] · [[layered-settings]] · [[project-trust-gate]] · [[subagent-hosting]]
