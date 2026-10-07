---
type: implementation
harness: codex
concept: tool-description-design
commit: 622e9e3696
files: [codex-rs/core/src/tools/handlers/apply_patch_spec.rs:20, codex-rs/core/src/tools/handlers/shell_spec.rs:20-266, codex-rs/core/src/tools/handlers/shell_spec.rs:339-345, codex-rs/core/src/tools/handlers/plan_spec.rs:44-47, codex-rs/core/src/tools/handlers/request_user_input_spec.rs:35-141, codex-rs/core/src/tools/handlers/tool_search_spec.rs:8-104, codex-rs/core/templates/search_tool/request_plugin_install_description.md:1-29, codex-rs/protocol/src/openai_models.rs:585-597, codex-rs/core/src/tools/code_mode/wait_spec.rs:53, codex-rs/code-mode-protocol/src/description.rs:16-133]
---
[[tool-description-design]] in [[codex]].

Style: **terse one-line descriptions + exact parameter docs**; behavioural policy lives in the per-model system prompt ([[per-model-system-prompt]]) — except for tools that are not always present (spawn_agent, goal tools, plugin install), which carry long policy text themselves. Many descriptions are **catalog-owned** and can be replaced per model.

## Mechanism
### Where guidance lives
- Placement rule (observed): always-present tools → guidance in the system prompt; optional tools → guidance in their own description (`codex-rs/core/src/tools/handlers/multi_agents_spec.rs:692-770` long spawn_agent text) — inverse of pi's generate-guidelines-from-tools ([[dynamic-tool-guidelines]]).
- `30ee24521b` 2025-08-13 "remove behavioral prompting from update_plan tool def" moved "Skip a plan when…", "Planning steps are … like tasks or TODOs" into the main prompt ([[tool-description-lies-about-async]] codex variant). When a tool is disabled, the prompt sections that mention it are stripped by literal heading match ([[plan-checklist-tool]]).
- **Catalog overrides**: model-catalog `ModelMessages.tools` (`ToolMessages{indirect_description_prefixes, send_user_message_async, multi_agent, code_mode, mcp_resources}`) can replace bundled descriptions/parameters (`codex-rs/protocol/src/openai_models.rs:585-597`), with fallback "Invalid catalog tool parameters; using bundled parameters" (`codex-rs/core/src/tools/code_mode/wait_spec.rs:53`). Repo text is only the bundled default; production models may see different text.

### Per-tool texts (current)
- `apply_patch`: "The `apply_patch` tool can be used to edit files. This is a FREEFORM tool, so do not wrap the patch in JSON." + Lark grammar (`apply_patch_spec.rs:20`) — the grammar replaced a 75-line prose spec (`8d637ae398` deleted `apply_patch_tool_instructions.md`) → [[patch-envelope-edit]].
- `exec_command`: "Runs a command in a PTY, returning output or a session ID for ongoing interaction." Params state defaults and ranges (`yield_time_ms` "Defaults to 10000 ms; effective range is 250-30000 ms", `max_output_tokens` "Defaults to 10000 tokens; larger requests may be capped by policy") — **limits as literals in the schema text**, not interpolated from constants (`shell_spec.rs:30-110`); `write_stdin`, `sandbox_permissions`, `justification`, `prefix_rule` docs (`:116-266`) → [[shell-execution]].
- Windows rules appended only when the **executor** is Windows (`windows_shell_guidance`, `shell_spec.rs:339-345`; `5ed294d49d`) → [[windows-destructive-cross-shell]].
- `update_plan`: "Updates the task plan. Provide an optional explanation and a list of plan items, each with a step and status. At most one step can be in_progress at a time." (`plan_spec.rs:44-47`).
- `request_user_input`: description templated from allowed modes, same policy drives runtime check (`998eb8f32b`: "Tool description and runtime availability checks should be driven by the same centralized mode policy") (`request_user_input_spec.rs:123-141`) → [[structured-user-question-tool]].
- `request_permissions`: "Request additional filesystem or network permissions from the user and wait for the client to grant a subset… Granted permissions apply automatically to later shell-like commands in the current turn, or for the rest of the session if the client approves them at session scope." (`shell_spec.rs:193-196`).
- `get_context_remaining` "Get the remaining tokens in the current context window." / `new_context` "Start a new context window. Does not clear, reset, or otherwise affect environment state." (`dac5f07403`, `87ab01834a`) → [[model-requested-context-reset]].
- `tool_search`: "# Tool discovery\n\nSearches over deferred tool metadata with BM25 and exposes matching tools for the next model call." + optional source listing ("You have access to tools from the following sources: - name: description", ≤ 512 KiB) + "For MCP tool discovery, always use `tool_search` instead of `list_mcp_resources` or `list_mcp_resource_templates`." (`tool_search_spec.rs:8,36-104`) → [[deferred-tool-loading]]. An older Apps-only template `codex-rs/core/templates/search_tool/tool_description.md` ("# Apps (Connectors) tool discovery…") has no Rust reference at `622e9e3696` (git grep; likely orphaned — unverified).
- `request_plugin_install`: "Use this ONLY when all of the following are true: The user explicitly asks to use a specific plugin…; `tool_search` is not available, or it has already been called and did not find…"; "Do not use this tool for adjacent capabilities, broad recommendations, or tools that merely seem useful."; "IMPORTANT: DO NOT call this tool in parallel with other tools." (`codex-rs/core/templates/search_tool/request_plugin_install_description.md:1-29`) → [[tool-name-semantics-misfire]].
- Code mode `exec`: full model-facing API contract (helpers, `tools.*` TypeScript declarations, deferred discovery via `await tools.tool_search(...)`) (`codex-rs/code-mode-protocol/src/description.rs:16-133`) → [[code-mode]].
- `clock.sleep`: "Pause execution for a specified duration. The sleep ends early when new input arrives for the active turn. Returns the elapsed wall-clock time." → [[wall-clock-tools]].
- `view_image`: "View a local image file from the filesystem when visual inspection is needed." (`codex-rs/core/src/tools/handlers/view_image_spec.rs:18-48`) → [[file-read-tool]].

### Removed descriptions
- Legacy `shell_command`: "Runs a shell command and returns its output. - Always set the `workdir` param when using the shell_command function. Do not use `cd` unless absolutely necessary." (`916fdc2a37` 2025-09-14 → removed `8a40095ea3` 2026-08-20); Windows PowerShell examples ("ls -a (show hidden): "Get-ChildItem -Force"") removed with raw `shell`/`local_shell` (`83decfa300`).

## Constants
| name | value | path:line |
|---|---|---|
| tool_search source description budget | 512 KiB | `codex-rs/core/src/tools/handlers/tool_search_spec.rs:8` |
| request_user_input questions / options / header | ≤3 (prefer 1) / 2-3 / ≤12 chars | `codex-rs/core/src/tools/handlers/request_user_input_spec.rs:35-72` |
| exec yield / output defaults stated in schema | 10000 ms (250-30000) / 10000 tokens | `codex-rs/core/src/tools/handlers/shell_spec.rs:30-61` |

## Evolution
- 2025-08-13 `30ee24521b` behavioural text out of update_plan.
- 2025-10-04 `4764fc1ee7` "FREEFORM tool, so do not wrap the patch in JSON".
- 2026-02-03 `998eb8f32b` mode-scoped request_user_input description.
- 2026-02-09 `becc3a0424` search_tool; 2026-02-13 `38c442ca7f` discovery guidance moved from a separate developer message into the tool description; 2026-03-12 `bc48b9289a` "Add mentions of connectors because model always think in connector terms in its CoT", "Suppress list_mcp_resources in favor of tool search".
- 2026-03-11 `ba5b94287e` `tool_suggest` → 2026-04-29 `8ce48f9968` tightened triggers + no-parallel → 2026-05-02 `f88701f5c8` renamed `request_plugin_install` ("Tool suggest still misfires when model needs tool_search… rephrase "suggestion" to "install"").
- 2026-03-19 `69750a0b5a` / 2026-04-23 `2e228969be` / 2026-08-28 `5ed294d49d` Windows rules.
- 2026-05-29 `1c55bb2702` "Improve built-in tool schema docs" — "Clarify default, omission, and bounded behavior across built-in tool schemas… Convert update_plan status to an enum".
- 2026-08-13 `8d637ae398` apply_patch prose instructions deleted.
- 2026-08-26 `ac644ed112` bounds no longer preserved in (foreign) input schemas ([[tool-schema-normalization]]).
- 2026-10-03 `58ca099b03` Code Mode tool discovery guidance stable across catalog changes; 2026-10-05 `402f5b6fdf` / `93f8e79fd2` incremental tool-catalog update texts ([[tool-loadout-stale-within-run]]).

## Quirks
- Limits appear as literals in schema text (10000 ms, 250-30000, 10000 tokens) — the same anti-pattern codex removed from its system prompt ([[prompt-states-stale-harness-limits]]) survives in parameter docs; they match the constants at `622e9e3696`.
- "Runs a command in a PTY" although the default is pipes.

## Versus pi
pi interpolates limits from constants, keeps descriptions byte-stable, and generates prompt guidelines from active tools ([[pi--tool-description-design]], [[pi--dynamic-tool-guidelines]]). codex: one-liners + model-owned prompt policy + catalog-replaceable tool text; the tool *name* is treated as the strongest description (renames to fix misfires).

## Failures
[[tool-name-semantics-misfire]] · [[windows-destructive-cross-shell]] · [[foreign-harness-tool-hallucination]] · [[tool-description-lies-about-async]] · [[malformed-tool-json-crashes]] · [[orchestrator-busy-polls-subagents]] · [[prompt-states-stale-harness-limits]]
