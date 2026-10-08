---
type: concept
stage: tool-design
tier: must-have
aliases: [DEFAULT_TOOL_NAMES, defaultTools, "--tools +x/-x", tool-selection-modifiers, createCodingTools, createReadOnlyTools, ToolRegistry, BuiltInTools, spec_plan, build_tool_router, add_core_tool_sources, experimental_supported_tools]
harnesses: [pi, opencode, codex]
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
- **Feature/model/provider-gated tool plan rebuilt every turn** — ✔ codex (`build_tool_router`, `codex-rs/core/src/tools/spec_plan.rs:123`): 2 core tools (`exec_command`/`write_stdin` + freeform `apply_patch`) plus ~20 tools gated by `Feature::*` flags, model-catalog fields (`shell_type`, `apply_patch_tool_type`, `supports_search_tool`, `experimental_supported_tools`), provider capabilities, root-vs-subagent and environment presence; no user allowlist grammar.
- Rich default partly adopted by codex: hosted web search (Cached), subagent tools, question tool on; todo tool **off** since `a9519cbcdd` 2026-08-31 (reversal toward pi) → [[web-tools]], [[ask-user-tool]], [[task-list-tool]].
- **No dedicated read/search/write tools at all; shell + one patch tool** (✔ codex; experimental `read_file`/`grep_files`/`list_dir` removed) → [[minimal-vs-rich-toolset]], [[shell-command-intent-parsing]].
- Evidence-driven pruning: remove tools that session history shows unused (codex `83decfa300` legacy shells "no active use"; `70807730f5` list_dir "nothing in the current model catalog advertises it").
- Policy kill switch: drop every core tool when a managed sandbox is required but the profile isn't managed (✔ codex `spec_plan.rs:1084-1095`).
- Duplicate-name policy: hard pre-sampling error on tool-name collisions (✔ codex `error_on_tool_collisions`).
- Rich default gated per request by client, provider, model id and env flags (opencode).
- Hide via permission rules (last match deny `*`) instead of an allowlist (opencode) → [[permission-ruleset]].
- Meta-tool for parallel calls (`batch`, 1–25 calls) tried and deleted (opencode) → [[parallel-tool-execution]].
- Swap tools per model family → [[model-specific-toolset]].

## Implementations
- [[pi--minimal-default-toolset|pi]] — 4 default tools, 8 built-ins, replaceable built-in extensions add codemode/tool_search/MCP; allowlist-or-modifiers grammar.
- [[codex--minimal-default-toolset|codex]] — contrasting design: per-turn gated plan; core = `exec_command`/`write_stdin` + `apply_patch`; full inventory of ~25 tools in the note.
- [[opencode--minimal-default-toolset|opencode]] — the opposite pole: ~14 built-ins on by default (question, shell, read, glob, grep, edit/write or apply_patch, task, webfetch, websearch, todowrite, skill) + flag-gated execute/lsp/plan_exit; many tools added then killed (batch, multiedit, list, todoread, codesearch, repo_*, task_status, plan_enter).

## Failures
- [[tool-allowlist-hides-mcp-tools]]

## Related
[[deferred-tool-loading]] · [[code-mode]] · [[mcp-integration]] · [[shell-execution]] · [[search-tools]] · [[dynamic-tool-guidelines]] · [[minimal-system-prompt]] · [[replaceable-builtin-extension]] · [[layered-settings]] · [[plugin-tools]] · [[minimal-vs-rich-toolset]] · [[patch-envelope-edit]] · [[shell-command-intent-parsing]] · [[feature-flag-stages]] · [[model-catalog]] · [[tool-wire-kinds]] · [[no-file-read-write-tools]] · [[removed-legacy-shell-tools]] · [[web-tools]] · [[task-list-tool]] · [[ask-user-tool]] · [[lsp-diagnostics-feedback]]

## Tradeoffs
- [[plan-mode-vs-none]]
- [[minimal-vs-rich-toolset]]
