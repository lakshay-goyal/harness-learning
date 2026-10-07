---
type: concept
stage: subagents
tier: candidate
aliases: [agent, "agent.<name>", default_agent, "mode: primary|subagent|all", native agent, hidden agent, AgentV2, Agent.generate, "opencode agent create", generate.txt, "{agent,agents}/**/*.md"]
harnesses: [opencode]
---
Named bundles of prompt, model, temperature, permission ruleset, step cap and mode (primary, subagent or all). The user switches between them, or the model delegates to them.

## Why
- Different jobs need different capability envelopes: a default executor, a read-only planner, a cheap read-only explorer for delegation, and internal side-call agents (title, summary, compaction) that must never call tools.
- Putting the permission ruleset on the profile makes mode switches and delegation the same mechanism: [[plan-mode]] is just an agent; a subagent is an agent with `mode: subagent`.
- Users extend the roster in config or markdown files without code; the delegating model needs each profile's description to choose.

## Design space
- **Profile contents**: prompt + model + sampling + permission + step cap + mode + color + hidden (opencode legacy) vs v2 drops agent `temperature`/`top_p` and renames `prompt` → `system` (`specs/v2/config.md:264-266`).
- **Mode**: primary (user-selectable), subagent (delegation target), all (opencode default for custom agents).
- **Built-ins**: build, plan (primary); general, explore (subagent); compaction, title, summary (hidden, `*: deny`) (opencode). v2 defines them inside a core plugin (`packages/core/src/plugin/agent.ts`).
- **Override**: user config merged after the agent's own rules, so last-match-wins lets config override built-in restrictions (opencode).
- **Authoring**: JSON config, markdown files with frontmatter (opencode `{agent,agents}/**/*.md`), or model-generated from a description (opencode `opencode agent create` → `Agent.generate`, a Claude Code "agent architect" prompt copy).
- **Advertising to the delegating model**: dynamic list of non-primary agents the caller may use, descriptions only (opencode task tool description).
- **pi**: no core profiles; the subagent example reads agent markdown with `tools`/`model` frontmatter ([[subagent-as-subprocess]]).

## Implementations
- [[opencode--agent-profiles|opencode]] — legacy `Agent.Service` state: 7 native agents, permission = `merge(defaults, agent, user)`, `cfg.agent.<name>` adds/disables/overrides, list sorted with `default_agent` first; v2 `AgentV2` with `permissions`, `system`, `steps`.

## Failures
- [[read-only-mode-bypass-via-subagent]]
- [[subagent-config-not-inherited]]
- [[model-initiated-mode-switch]]

## Tradeoffs
- [[plan-mode-vs-none]]
- [[builtin-subagents-vs-none]]

## Related
[[plan-mode]] · [[task-owned-subagent]] · [[permission-ruleset]] · [[step-budget-limit]] · [[auxiliary-model-calls]] · [[subagent-as-subprocess]] · [[layered-settings]] · [[opencode]]
