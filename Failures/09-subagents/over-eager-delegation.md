---
type: failure
concepts: [task-owned-subagent]
harnesses: [codex]
---
**Symptom** — The model spawned sub-agents for ordinary "be thorough / investigate" requests and because role descriptions looked like invitations.

**Root cause** — A tool description that explains *how* to use a powerful tool reads as encouragement; role guidance read as authorization.

**Fix · [[codex]]**
- `8f8a0f55ce` "spawn prompt (#14362)" + `367a8a2210` "Clarify spawn agent authorization (#14432)" 2026-03-11: "Do not spawn sub-agents unless the user or applicable AGENTS.md/skill instructions explicitly ask for sub-agents, delegation, or parallel agent work. Requests for depth, thoroughness, research, investigation, or detailed codebase analysis do not count as permission to spawn." (`codex-rs/core/src/tools/handlers/multi_agents_spec.rs:738-739`) + "Agent-role guidance … never authorizes spawning by itself"; regression test asserts the strings. V2 places the no-spawn instruction last (`127224cacc` 2026-06-15).
- Opposite mode is opt-in via superseding developer message: `PROACTIVE_MULTI_AGENT_MODE_TEXT` (`codex-rs/prompts/src/model_messages/multi_agent.rs:47`), only at `Ultra` effort (`codex-rs/core/src/session/multi_agents.rs:93-104`).

**Lesson** — State the authorization condition for powerful tools explicitly and enumerate the near-miss phrasings that do not satisfy it.

Related: [[task-owned-subagent]] · [[agent-roles]] · [[imperative-guideline-over-compliance]] · [[tool-description-design]] · [[codex--task-owned-subagent|codex]]
