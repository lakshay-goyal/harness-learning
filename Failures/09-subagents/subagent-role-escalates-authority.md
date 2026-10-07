---
type: failure
concepts: [agent-profiles]
harnesses: [codex]
---
**Symptom** — Agent role config files were full config layers and could change a child's permissions, providers, endpoints, MCP servers and notifications — delegation as a privilege-escalation path.

**Fix · [[codex]]**
- `1a6e07a4fe` 2026-08-18 "Restrict agent roles to bounded configuration overrides" — "Agent roles should customize a child agent without expanding the authority or changing the provider configuration inherited from its parent session… reject symlinked user role files". Allowlist: developer_instructions, model, reasoning effort/summary, verbosity, personality, service tier; features only *off* (ShellTool, Apps, Plugins, MemoryTool, RequestPermissionsTool); skills only disabled (`codex-rs/core/src/agent/role.rs:69-127`).

**Lesson** — Delegation presets should only narrow capability; validate role layers against an allowlist.

Related: [[agent-profiles]] · [[subagent-config-inheritance]] · [[layered-settings]] · [[untrusted-repo-loads-executable-config]] · [[codex--agent-profiles|codex]]
