---
type: implementation
harness: opencode
concept: minimal-default-toolset
commit: ecc4916b5a
files: [packages/opencode/src/tool/registry.ts:118-119, packages/opencode/src/tool/registry.ts:196-252, packages/opencode/src/tool/registry.ts:265-300, packages/opencode/src/effect/runtime-flags.ts:22-56, packages/opencode/src/permission/index.ts:204-219, packages/opencode/src/session/llm/request.ts:210-216, packages/opencode/src/session/tools.ts:388, packages/core/src/tool/builtins.ts:26-48]
---
[[minimal-default-toolset]] in [[opencode]]. opencode is the opposite pole: a rich default.

## Mechanism
### Legacy runtime: built-in catalog (registry order)
| tool | role | gate |
|---|---|---|
| `invalid` | sink for broken calls, "Do not use" | registered, never in `activeTools` → [[tool-argument-repair]] |
| `question` | ask user | interactive clients or flag → [[ask-user-tool]] |
| `shell` | bash/pwsh/cmd | always → [[shell-execution]] |
| `read` | files, dirs, images, PDF | always → [[file-read-tool]] |
| `glob`, `grep` | ripgrep search | always → [[search-tools]] |
| `edit`, `write` | string replace / overwrite | hidden for patch-format models → [[search-replace-edit]] |
| `task` | subagent; description lists permitted agents | always → [[task-owned-subagent]] |
| `webfetch`, `websearch` | web | websearch provider/flag-gated → [[web-tools]] |
| `todowrite` | todo list | always → [[task-list-tool]] |
| `skill` | load skill body | always → [[skill-progressive-disclosure]] |
| `apply_patch` | patch envelope | `gpt-*` ids only → [[patch-envelope-edit]] |
| `execute` | code mode | `OPENCODE_EXPERIMENTAL_CODE_MODE` and non-empty MCP catalog → [[code-mode]] |
| `lsp` | navigation | `OPENCODE_EXPERIMENTAL_LSP_TOOL` → [[lsp-diagnostics-feedback]] |
| `plan_exit` | leave plan mode | `OPENCODE_EXPERIMENTAL_PLAN_MODE` + client `cli` → [[plan-mode]] |
(`packages/opencode/src/tool/registry.ts:207-252`.)
- Custom tools: `{tool,tools}/*.{js,ts}` in every config dir (`default` export → file name, else `file_export`) plus plugin `tool` maps (`registry.ts:183-204`) → [[plugin-tools]].
- Per request `tools(input)`: provider/model filtering (`registry.ts:290-300`) → [[model-specific-toolset]]; `tool.definition` plugin hook may rewrite description/schema; task description gets "Available agent types and the tools they have access to:" (`:277`).
- Hiding = permission: a tool is dropped when the last matching rule is `deny` with pattern `*` (`packages/opencode/src/permission/index.ts:204-219`); per-message `tools: {name:false}` also drops (`packages/opencode/src/session/llm/request.ts:210-216`) → [[permission-ruleset]].
- Code mode on → MCP tools not declared directly (`packages/opencode/src/session/tools.ts:388`).
### v2 runtime
- Static `BuiltInTools`: apply_patch, bash, edit, glob, grep, question, read, skill, todowrite, webfetch, websearch, write (`packages/core/src/tool/builtins.ts:31-48`); TODO "Port the remaining launch-follow-up leaves deliberately: edit fuzzy parity, task, LSP, repo_clone, repo_overview, plan_exit, and Rune/code mode" (`:26-29`).

## Constants
| name | value | path:line |
|---|---|---|
| default client | `"cli"` | `packages/opencode/src/effect/runtime-flags.ts:56` |
| batch size (removed) | 1–25 | `463318486f^:packages/opencode/src/tool/batch.ts:38` |

## Evolution (removed tools)
- `multiedit`: registry-commented 2025-06-05 `35b03e4cb3`, deleted 2026-04-21 `2486621ca1` "kill unused tool".
- `list`: 2025-12-25 `f397c92ddf`; files 2026-04-15 `d2ea6700aa`; superseded by `read` on directories (`6b4d617df0`).
- `lsp-hover`, `lsp-diagnostics`: 2025-12-26 `1e2ef07c97` "kill some unused tools"; diagnostics now pushed into edit results.
- `batch`: experimental 2025-11-15 `1056b36eae` ("Executes multiple independent tool calls concurrently… USING THE BATCH TOOL WILL MAKE THE USER HAPPY.", 1–25 calls, no nesting — `463318486f^:packages/opencode/src/tool/batch.txt:1-12`; limit 10 → 25 `673e79f457` 2026-01-19); deleted in the 2026-04-07 tool-system refactor `463318486f`. No stated reason (unverified); native parallel calls cover it → [[parallel-tool-execution]].
- `todoread`: 2026-02-02 `1275c71a63`, dead code 2026-03-25 `77fc88c8ad`.
- `patch` → `apply_patch` 2026-01-17 `b7ad6bd839`.
- `plan_enter`: disabled 2026-02-24 `fa559b0385` "to prevent unintended mode switches", internals later deleted → [[plan-mode]].
- `codesearch`: 2025-11-10 `3f5acc3dff` → "rm broken" 2026-04-29 `6aa8e894b1` → re-added by scout → removed 2026-05-12 `2481dde36d`.
- `repo_clone`/`repo_overview` + `scout` agent: 2026-05-09 `40d5ea1cf1` → 2026-06-02 `a639fe7a08`; replaced by config references → [[project-references]].
- `task_status`: removed 2026-05-25 `dabf2dc013` "remove the need for polling".

## Quirks / drift
- The `list` permission key outlives its tool (`packages/opencode/src/agent/agent.ts:204`).

Contrast: pi ships read/bash/edit/write on and everything else opt-in → [[pi--minimal-default-toolset|pi]].
