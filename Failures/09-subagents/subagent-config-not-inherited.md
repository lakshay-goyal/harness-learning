---
type: failure
concepts: [subagent-as-subprocess, subagent-config-inheritance]
harnesses: [pi, opencode, codex]
---
**Symptom** — Subagents without their own `model` ignored the dispatching session's active model and thinking level (#7897; tools were never inherited — each agent's `tools` frontmatter applies); YAML array-form `tools: [read, bash]` in agent frontmatter was rejected (#7598); agents resolved from hardcoded dirs instead of the configured agent dir (#1559).

**Root cause** — Spawn-time configuration was implicit: the child CLI started with its own defaults; frontmatter parser assumed one scalar shape; path computed independently of `getAgentDir()`.

**Fix · [[pi]]**
- `7390f830d` 2026-02-26 — `getAgentDir()` for user agents path (#1559; 0.55.1).
- `e3798ca91` 2026-08-11 — inherit dispatcher `provider/id` model and, when the agent sets no model, `--thinking` level (`packages/coding-agent/examples/extensions/subagent/index.ts:301-306,485-488`) (#7897).
- `d268454e9` 2026-08-14 — accept string or array `tools`; bad value → no tools rather than throwing during discovery (`subagent/agents.ts:42-59`) (#7598).

**Fix · [[codex]]** (in-process children; one helper builds child config from the parent's live turn)
- Symptoms/fixes: split sandbox policies not synced to children (`7f571396c8` 2026-03-13); role ignored under profiles (`33acc1e65f` 2026-03-16); persisted approvals not shared — "If a subagent requests approval, and the user persists that approval to the execpolicy, it should (by default) propagate" (`84f4e7b39d` 2026-03-17); forked children lost parent model config (`776246c3f5` 2026-04-13); configured subagent model defaults ignored (`21c37fb374` 2026-07-16); reloaded V2 agents lost model provider (`d1f14e31a7` 2026-08-04); service tier not following the root (`dc2ccc6843` 2026-08-28); pending environments not inherited (`3749d1eff7` 2026-09-28).
- Fix pattern: `build_agent_shared_config` + `apply_spawn_agent_runtime_overrides` derive the child from the live turn (model, provider, effort, instructions, approvals, cwd, permission profile) — doc warns cloning stale config "can send the child agent out with the wrong provider or runtime policy" (`codex-rs/core/src/agent/child_config.rs`); environments from the requesting step (`codex-rs/core/src/agent/control/spawn.rs:726-736`).
**Fix · [[opencode]]** — both directions of the spawn-time contract:
- child → parent leak: user-invoked subtasks changed the main session's selected model/agent (`8b08d340ac` 2026-01-15, `08b94a6890` 2026-01-16 "keep primary model after subagent runs"); `session_complete` hook fired for child sessions (`a524fc545c` 2025-07-20); `--continue` resumed a subagent session (`a2db58f125` 2025-08-19).
- parent → child: nested subagents ignored their own `task` permission (`5d37e58d34` 2026-01-13); parent session `deny` + `external_directory` rules now copied to the child (`d7701dbfb6` 2026-04-30; `packages/opencode/src/agent/subagent-permissions.ts:14-27`); child model defaults to the parent message's model and variant unless the agent sets one (`packages/opencode/src/tool/task.ts:181-184`). Agent-level restrictions deliberately not inherited ([[read-only-mode-bypass-via-subagent]]).

**Lesson** — Spawn-time config inheritance must be explicit, derived from the parent's *effective runtime* state (not persisted config), and tested; tolerate both spellings of user-authored config.
Related: [[subagent-as-subprocess]] · [[model-resolution]] · [[thinking-level-abstraction]] · [[pi--subagent-as-subprocess|pi]] · [[opencode--task-owned-subagent|opencode]] · [[task-owned-subagent]]

Related: [[subagent-as-subprocess]] · [[model-resolution]] · [[thinking-level-abstraction]] · [[pi--subagent-as-subprocess|pi]] · [[subagent-config-inheritance]] · [[subagent-model-downgrade]] · [[codex--subagent-config-inheritance|codex]]
