---
type: concept
stage: tool-design
tier: candidate
aliases: [DEFAULT_TOOL_NAMES, defaultTools, "--tools +x/-x", tool-selection-modifiers, createCodingTools, createReadOnlyTools, ToolRegistry, BuiltInTools]
harnesses: [pi, opencode]
---
Ship a tiny, general toolset on by default (read / shell / edit / write) and make every specialized tool opt-in through an allowlist or additive modifiers.

## Why
- Every declared tool costs prompt tokens on every request and competes for the model's attention; general tools (a shell) subsume many specialized ones.
- Without a selection grammar, users can only replace the whole set, so adding one tool means re-listing defaults; mixing allowlist and modifiers is ambiguous.
- A blunt allowlist can accidentally cut off tools from other sources (MCP) — [[tool-allowlist-hides-mcp-tools]].

## Design space
- **Tiny general default + opt-in extras** — pi: read/bash/edit/write on; grep/find/ls/powershell/codemode/tool_search off.
- Rich default (search, web, todo, subagent, plan) — rejected by pi by stance ([[no-todo-tool]], [[no-web-tools]], [[no-subagents-core]], [[no-plan-mode]]).
- Read-only profile as a named bundle (pi: `createReadOnlyTools` = read/grep/find/ls).
- Selection grammar: plain allowlist vs `+name/-name` modifiers appended across settings layers (pi: both, mutually exclusive per list) vs denylist (`--exclude-tools`).
- Specialized tools auto-activated by configuration (pi: codemode/tool_search auto-on when MCP exposure needs them).
- Registered-but-undeclared tools as an alternative to "off" → [[deferred-tool-loading]].
- Rich default gated per request by client, provider, model id and env flags (opencode).
- Hide via permission rules (last match deny `*`) instead of an allowlist (opencode) → [[permission-ruleset]].
- Meta-tool for parallel calls (`batch`, 1–25 calls) tried and deleted (opencode) → [[parallel-tool-execution]].
- Swap tools per model family → [[model-specific-toolset]].

## Implementations
- [[pi--minimal-default-toolset|pi]] — 4 default tools, 8 built-ins, replaceable built-in extensions add codemode/tool_search/MCP; allowlist-or-modifiers grammar.
- [[opencode--minimal-default-toolset|opencode]] — the opposite pole: ~14 built-ins on by default (question, shell, read, glob, grep, edit/write or apply_patch, task, webfetch, websearch, todowrite, skill) + flag-gated execute/lsp/plan_exit; many tools added then killed (batch, multiedit, list, todoread, codesearch, repo_*, task_status, plan_enter).

## Failures
- [[tool-allowlist-hides-mcp-tools]]

## Tradeoffs
- [[plan-mode-vs-none]]
- [[minimal-vs-rich-toolset]]

## Related
[[deferred-tool-loading]] · [[code-mode]] · [[mcp-integration]] · [[shell-execution]] · [[search-tools]] · [[dynamic-tool-guidelines]] · [[minimal-system-prompt]] · [[replaceable-builtin-extension]] · [[layered-settings]] · [[plugin-tools]] · [[web-tools]] · [[task-list-tool]] · [[ask-user-tool]] · [[lsp-diagnostics-feedback]] · [[patch-envelope-edit]]
