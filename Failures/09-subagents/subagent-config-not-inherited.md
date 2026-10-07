---
type: failure
concepts: [subagent-as-subprocess]
harnesses: [pi]
---
**Symptom** — Subagents without their own `model` ignored the dispatching session's active model and thinking level (#7897; tools were never inherited — each agent's `tools` frontmatter applies); YAML array-form `tools: [read, bash]` in agent frontmatter was rejected (#7598); agents resolved from hardcoded dirs instead of the configured agent dir (#1559).

**Root cause** — Spawn-time configuration was implicit: the child CLI started with its own defaults; frontmatter parser assumed one scalar shape; path computed independently of `getAgentDir()`.

**Fix · [[pi]]**
- `7390f830d` 2026-02-26 — `getAgentDir()` for user agents path (#1559; 0.55.1).
- `e3798ca91` 2026-08-11 — inherit dispatcher `provider/id` model and, when the agent sets no model, `--thinking` level (`packages/coding-agent/examples/extensions/subagent/index.ts:301-306,485-488`) (#7897).
- `d268454e9` 2026-08-14 — accept string or array `tools`; bad value → no tools rather than throwing during discovery (`subagent/agents.ts:42-59`) (#7598).

**Lesson** — Spawn-time config inheritance must be explicit and tested; tolerate both spellings of user-authored config.

Related: [[subagent-as-subprocess]] · [[model-resolution]] · [[thinking-level-abstraction]] · [[pi--subagent-as-subprocess|pi]]
