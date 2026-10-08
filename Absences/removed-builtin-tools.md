---
type: absence
harnesses: [opencode]
---
# removed-builtin-tools

opencode's default toolset grew rich, then shed tools that were redundant, unused or broken. Index of removals (the ones with a design lesson have their own notes).

**What's missing** (each was built in once)
| tool | added | removed | reason (from history) | replaced by |
|---|---|---|---|---|
| `list` (ls) | (≤ `f3da73553c` 2025-05-30) | registry `f397c92ddf` 2025-12-25; files `d2ea6700aa` 2026-04-15 | `read` handles directories since `6b4d617df0` (2026-02-11) | `read` on a dir; the `list` permission key survives in `explore` (`packages/opencode/src/agent/agent.ts:204`) |
| `multiedit` | — | commented out `35b03e4cb3` 2025-06-05; deleted "unused" `2486621ca1` 2026-04-21 | unused | repeated `edit` / `apply_patch` |
| `batch` | `1056b36eae` 2025-11-15 (experimental) | `463318486f` 2026-04-07 | none stated (unverified) | native parallel calls, code mode → [[no-batch-tool]] |
| `todoread` | — | disabled `1275c71a63` 2026-02-02; deleted `77fc88c8ad` 2026-03-25 | `todowrite` already echoes the full list | `todowrite` → [[task-list-tool]] |
| `lsp-hover`, `lsp-diagnostics` | — | `1e2ef07c97` 2025-12-26 "kill some unused tools" | unused | diagnostics pushed into edit results → [[lsp-diagnostics-feedback]] |
| `codesearch` (Exa Code API) | `3f5acc3dff` 2025-11-10 | `6aa8e894b1` 2026-04-29 "rm broken codesearch tool"; re-added by accident in `40d5ea1cf1`; removed again `2481dde36d` 2026-05-12 | broken | `websearch`, `explore` → [[web-tools]] |
| `repo_clone`, `repo_overview` + `scout` agent | `40d5ea1cf1` 2026-05-09 | `a639fe7a08` 2026-06-02 | none stated | config `references` + read whitelisting (`packages/opencode/src/agent/agent.ts:102-112`) → [[project-references]] |
| `task_status` | `22de34c4de` 2026-05-14 | `dabf2dc013` 2026-05-25 | polling | push completion → [[no-background-task-polling]] |
| `plan_enter` | `0a3c72d678` 2026-01-13 | disabled `fa559b0385` 2026-02-24 | unintended mode switches | user switches agent → [[no-model-initiated-plan-entry]] |
| `patch` (old) | — | `b7ad6bd839` 2026-01-17 | superseded | `apply_patch` for GPT models → [[patch-envelope-edit]] |

- v2 Core still lists `repo_clone`/`repo_overview` as "launch-follow-up leaves" (`packages/core/src/tool/builtins.ts:26-28`), so the scout idea is parked, not dead.
- Not a removal but a guard that went away: [[no-read-before-write-guard]].

**Evidence of decision**
- Governance: "any UI or core product feature must go through a design review with the core team before implementation" (`CONTRIBUTING.md:13`). Readily merged: bug fixes, LSPs/formatters, LLM performance, providers (`CONTRIBUTING.md:3-11`).

**Implication**
- The pattern: ship a tool, watch usage, delete what the model doesn't pick or what a broader tool subsumes (`read` ⊃ `list`, `todowrite` ⊃ `todoread`, edit-result diagnostics ⊃ diagnostics tool). Tool count stays ~14 by default despite a "rich" stance (`packages/opencode/src/tool/registry.ts:230-248`).
- Contrast pi: 4 default tools, and "No built-in agent tool has ever been deleted" ([[Absences]]) → [[minimal-vs-rich-toolset]].

Related: [[minimal-default-toolset]] · [[model-specific-toolset]] · [[tool-description-design]] · [[opencode]] · [[Absences]]
