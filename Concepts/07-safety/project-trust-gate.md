---
type: concept
stage: permissions
tier: candidate
aliases: [project trust, /trust, trust.json, defaultProjectTrust, project_trust, --approve, --no-approve, isProjectTrusted, project-trust-gating]
harnesses: [pi]
---
Explicit, remembered trust decision per directory that gates whether repo-supplied *executable or behavior-changing* config (settings, plugins, packages, prompt overrides, skills) is loaded — an input-loading guard, not an action guard.

## Why
- Coding agents auto-load project-local config; without a gate, cloning a repo and starting the agent executes the repo's plugins with user privileges ([[untrusted-repo-loads-executable-config]]).
- Scoping mistakes are easy: treating the agent's own config dir as project input when run from `$HOME` ([[trust-scope-includes-agent-config-dir]]).
- Opt-in safety plugins that read project files outside the gated list re-open the hole ([[repo-config-disables-sandbox-plugin]]).

## Design space
- **What is gated**: executable plugins only / all project config / also instruction files (AGENTS.md). pi gated instruction files for 4 days (`89a92207f` → `5cb4f597f`) then ungated them: prompt injection is unpreventable anyway, so trust only guards settings + executables.
- **Trigger**: any project dir vs. only when trust-requiring resources exist (pi: auto-trusted when none exist).
- **Decision sources & precedence**: CLI flag → plugin decision → saved per-dir decision (nearest ancestor) → global default → interactive prompt → deny (pi).
- **Scope of a decision**: exact dir, parent dir (inherits to children), session-only (pi offers all three).
- **Non-interactive default**: deny unless `always`/flag (pi) vs allow.
- **Plugin-decidable trust** (`project_trust` event from global/CLI extensions only) — pi; project plugins obviously cannot vote.
- **Action-time trust** (per-command approvals) — out of scope; see [[tool-call-gate]].

## Implementations
- [[pi--project-trust-gate|pi]] — `.pi/{settings.json,mcp.json,extensions,skills,prompts,themes,SYSTEM.md,APPEND_SYSTEM.md}` or ancestor `.agents/skills` trigger a trust decision; `~/.pi/agent/trust.json`; `defaultProjectTrust` ask|always|never; AGENTS.md/CLAUDE.md ungated.

## Failures
- [[untrusted-repo-loads-executable-config]]
- [[trust-scope-includes-agent-config-dir]]
- [[repo-config-disables-sandbox-plugin]]

## Related
[[context-file-hierarchy]] · [[layered-settings]] · [[runtime-plugin-loading]] · [[harness-package-distribution]] · [[mcp-integration]] · [[tool-call-gate]] · [[no-prompt-injection-defense]] · [[supply-chain-pinning]]
