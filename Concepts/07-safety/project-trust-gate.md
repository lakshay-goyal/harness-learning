---
type: concept
stage: permissions
tier: must-have
aliases: [project trust, /trust, trust.json, defaultProjectTrust, project_trust, --approve, --no-approve, isProjectTrusted, project-trust-gating, trust_level, ProjectTrustDecision, "[projects] table", trusted_hash, --dangerously-bypass-hook-trust]
harnesses: [pi, codex]
---
Explicit, remembered trust decision per directory that gates whether repo-supplied *executable or behavior-changing* config (settings, plugins, packages, prompt overrides, skills) is loaded — an input-loading guard, not an action guard.

## Why
- Coding agents auto-load project-local config; without a gate, cloning a repo and starting the agent executes the repo's plugins with user privileges ([[untrusted-repo-loads-executable-config]]).
- Scoping mistakes are easy: treating the agent's own config dir as project input when run from `$HOME` ([[trust-scope-includes-agent-config-dir]]).
- Opt-in safety plugins that read project files outside the gated list re-open the hole ([[repo-config-disables-sandbox-plugin]]).

## Design space
- **What is gated**: executable plugins only / all project config / also instruction files (AGENTS.md). pi gated instruction files for 4 days (`89a92207f` → `5cb4f597f`) then ungated them: prompt injection is unpreventable anyway, so trust only guards settings + executables. ✔ codex gates project config, hooks, exec-policy rules **and** AGENTS.md (`bd19459358`), and avoids running repo-controlled git/PATH helpers before trust.
- **Trigger**: any project dir vs. only when trust-requiring resources exist (pi: auto-trusted when none exist). codex: prompt for undecided local projects (auto-trust tried and reverted same day `1e59dc5bda` → `17801b4206`); projectless dirs not persisted.
- **Decision sources & precedence**: CLI flag → plugin decision → saved per-dir decision (nearest ancestor) → global default → interactive prompt → deny (pi).
- **Scope of a decision**: exact dir, parent dir (inherits to children), session-only (pi offers all three); repo root with linked worktrees inheriting, resolved from `.git` on disk without running git (✔ codex).
- **Non-interactive default**: deny unless `always`/flag (pi) vs allow.
- **Plugin-decidable trust** (`project_trust` event from global/CLI extensions only) — pi; project plugins obviously cannot vote.
- **Trust drives runtime defaults** (✔ codex): trusted → approval `on-request` + workspace-write; untrusted → prompt for every command not allowed by a rule; undecided → read-only profile. pi: trust only gates loading.
- **Per-artifact trust**: content-hash trust per hook definition, unmanaged hooks blocked until reviewed (✔ codex `trusted_hash`).
- **Action-time trust** (per-command approvals) — out of scope; see [[tool-call-gate]], [[approval-policy-modes]].
- **No gate, declared out of scope** (opencode: project config, `.opencode/plugin(s)` and `.opencode/tool(s)` execute on open; "Malicious config files … not an attack vector", `SECURITY.md:33`; only lever `OPENCODE_DISABLE_PROJECT_CONFIG`).

## Implementations
- [[pi--project-trust-gate|pi]] — `.pi/{settings.json,mcp.json,extensions,skills,prompts,themes,SYSTEM.md,APPEND_SYSTEM.md}` or ancestor `.agents/skills` trigger a trust decision; `~/.pi/agent/trust.json`; `defaultProjectTrust` ask|always|never; AGENTS.md/CLAUDE.md ungated.
- [[codex--project-trust-gate|codex]] — `[projects."<root>"] trust_level`; untrusted ⇒ project layer `disabled_reason` (config, hooks, exec policies) + AGENTS.md skipped; trust selects approval/sandbox defaults; hook `trusted_hash`.

## Failures
- [[untrusted-repo-loads-executable-config]]
- [[trust-scope-includes-agent-config-dir]]
- [[repo-config-disables-sandbox-plugin]]
- [[agent-writes-its-own-escalation-config]] (codex: agent writing trusted `.codex/` from inside the sandbox)

## Related
[[context-file-hierarchy]] · [[layered-settings]] · [[runtime-plugin-loading]] · [[harness-package-distribution]] · [[mcp-integration]] · [[tool-call-gate]] · [[no-prompt-injection-defense]] · [[supply-chain-pinning]] · [[protected-workspace-metadata]] · [[approval-policy-modes]] · [[command-rule-policy]]

## Tradeoffs
- [[permission-prompts-vs-none]]
