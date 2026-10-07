---
type: concept
stage: subagents
tier: must-have
aliases: [agent, "agent.<name>", default_agent, "mode: primary|subagent|all", native agent, hidden agent, AgentV2, Agent.generate, "opencode agent create", generate.txt, "{agent,agents}/**/*.md", subagent-naming, AgentRoleConfig, AgentRoleToml, "[agents.<name>]", agent_type, explorer role, worker role, awaiter role, codex-agent-roles, agent_names.txt, agent_nickname, nickname_candidates, nickname pool, agent-roles]
harnesses: [opencode, codex]
---
Named bundles of prompt, model, temperature, permission ruleset, step cap and mode (primary, subagent or all). The user switches between them, or the model delegates to them.

## Why
- Different jobs need different capability envelopes: a default executor, a read-only planner, a cheap read-only explorer for delegation, and internal side-call agents (title, summary, compaction) that must never call tools.
- Putting the permission ruleset on the profile makes mode switches and delegation the same mechanism: [[plan-mode]] is just an agent; a subagent is an agent with `mode: subagent`.
- Users extend the roster in config or markdown files without code; the delegating model needs each profile's description to choose.
- Delegation quality depends on telling the parent *when* each preset fits (explorer vs worker) and the child *how* to behave (file ownership, shared workspace).
- A role file that is a full config layer can change permissions, providers, endpoints, MCP servers — a privilege escalation path via delegation ([[subagent-role-escalates-authority]]).
- Role descriptions read as invitations to spawn ([[over-eager-delegation]]).
- Users need to tell parallel children apart in the UI.

## Design space
- **Profile contents**: prompt + model + sampling + permission + step cap + mode + color + hidden (opencode legacy) vs v2 drops agent `temperature`/`top_p` and renames `prompt` → `system` (`specs/v2/config.md:264-266`).
- **Mode**: primary (user-selectable), subagent (delegation target), all (opencode default for custom agents).
- **Built-ins**: build, plan (primary); general, explore (subagent); compaction, title, summary (hidden, `*: deny`) (opencode). v2 defines them inside a core plugin (`packages/core/src/plugin/agent.ts`).
- **Override**: user config merged after the agent's own rules, so last-match-wins lets config override built-in restrictions (opencode).
- **Authoring**: JSON config, markdown files with frontmatter (opencode `{agent,agents}/**/*.md`), or model-generated from a description (opencode `opencode agent create` → `Agent.generate`, a Claude Code "agent architect" prompt copy).
- **Advertising to the delegating model**: dynamic list of non-primary agents the caller may use, descriptions only (opencode task tool description).
- **pi**: no core profiles; the subagent example reads agent markdown with `tools`/`model` frontmatter ([[subagent-as-subprocess]]).
- **Definition**: markdown + frontmatter agents (pi [[subagent-as-subprocess]]) · TOML `[agents.<name>]` with description, `config_file`, `nickname_candidates` + built-ins (✔ codex).
- **Override scope**: anything · bounded allowlist (instructions, model, effort/summary, verbosity, personality, service tier; features only *off* for a few; skills only disabled) (✔ codex).
- **Exposure**: always in schema · `agent_type` hidden unless roles configured (✔ codex).
- **Built-ins**: none (pi core) · `default`, `explorer` (empty config, "spawn explorers in parallel and trust their results"), `worker` (explicit file ownership, "not alone in the codebase"); `awaiter` disabled (✔ codex).
- **Fork interplay**: forbid role on full-history forks (codex V1) · allow (codex V2).
- **Naming**: ids · random unique nicknames from a 101-name pool with ordinal suffixes after exhaustion (✔ codex).
- (folded from `agent-roles`, codex framing) Named child presets (description + bounded config layer + nickname pool) that the parent model selects via an `agent_type` argument; roles may only narrow authority, never widen it. Children also get human-readable nicknames from a bundled pool.

## Implementations
- [[opencode--agent-profiles|opencode]] — legacy `Agent.Service` state: 7 native agents, permission = `merge(defaults, agent, user)`, `cfg.agent.<name>` adds/disables/overrides, list sorted with `default_agent` first; v2 `AgentV2` with `permissions`, `system`, `steps`.
- [[codex--agent-profiles|codex]] — `codex-rs/core/src/agent/role.rs` + `codex-rs/agent-roles`; bounded overrides, symlinked user role files rejected; nickname pool `agent_names.txt`.

## Failures
- [[read-only-mode-bypass-via-subagent]]
- [[subagent-config-not-inherited]]
- [[model-initiated-mode-switch]]
- [[subagent-role-escalates-authority]]
- [[over-eager-delegation]]

## Tradeoffs
- [[plan-mode-vs-none]]
- [[builtin-subagents-vs-none]]

## Related
[[plan-mode]] · [[task-owned-subagent]] · [[permission-ruleset]] · [[step-budget-limit]] · [[auxiliary-model-calls]] · [[subagent-as-subprocess]] · [[layered-settings]] · [[opencode]]
[[subagent-config-inheritance]] · [[in-process-subagent-threads]] · [[task-owned-subagent]] · [[subagent-as-subprocess]] · [[layered-settings]] · [[project-trust-gate]] · [[builtin-subagents-vs-none]]
