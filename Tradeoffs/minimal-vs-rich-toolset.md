---
type: tradeoff
concepts: [minimal-default-toolset, model-specific-toolset, deferred-tool-loading, replaceable-builtin-extension]
harnesses: [pi, opencode]
---
# minimal-vs-rich-toolset

**Axis**: how many tools are active by default, and who adds the rest?

| option | pi | opencode | evidence |
|---|---|---|---|
| Default active tools | 4: `read, bash, edit, write` | ~14: `invalid, question†, bash, read, glob, grep, edit, write, task, webfetch, todowrite, websearch†, skill, apply_patch†` (+ flag-gated `execute`, `lsp`, `plan_exit`) | pi: `packages/coding-agent/src/core/settings-manager.ts:215`; opencode: `packages/opencode/src/tool/registry.ts:230-248` |
| Search tools | built-in but off (`grep/find/ls`); prompt says "Use bash for … ls, rg, find" | on (`glob`, `grep`; `list` folded into `read`) | pi: `packages/coding-agent/src/core/system-prompt.ts:108-116` → [[search-tools]] |
| Extra capability path | extensions; codemode / tool_search / MCP as replaceable built-in extensions, not declared by default | config/plugin tools from `.opencode/tool(s)`, permission-driven hiding per agent | pi → [[replaceable-builtin-extension]], [[deferred-tool-loading]]; opencode `packages/opencode/src/permission/index.ts:204-219` |
| Toolset varies by model | no (one set; Codex bridge told GPT "APPLY_PATCH DOES NOT EXIST", `1650041a6`) | yes: `apply_patch` replaces `edit`/`write` for `gpt-` models | `b7ad6bd839` → [[model-specific-toolset]] |
| Tool churn | "No built-in agent tool has ever been deleted" | many removals (list, multiedit, batch, todoread, lsp-*, codesearch, scout, task_status, plan_enter) | [[Absences]]; [[removed-builtin-tools]] |
| Governance | "**pi's core is minimal** … PRs that bloat the core will likely be rejected" (`CONTRIBUTING.md:7-9`) | "any UI or core product feature must go through a design review" (`CONTRIBUTING.md:13`) | — |

**When each wins**
- **Minimal (pi)**: frontier models that do fine with bash; stable, cacheable tool block; tiny system prompt; extension authors get the same API as built-ins. Cost: every capability (subagents, todos, web) is an install away, and models trained on richer toolsets reach for tools that don't exist.
- **Rich (opencode)**: out-of-box parity with Claude Code/Codex habits and weaker models that need explicit tools. Cost: more surface to prune (opencode's removal log is the evidence), more per-model special-casing, and tool descriptions that drift from what the mechanism enforces ([[tool-description-design]]).

Related: [[minimal-default-toolset]] · [[edit-tool-variants]] · [[web-tools-vs-none]] · [[todo-tool-vs-none]] · [[pi]] · [[opencode]] · [[Tradeoffs]]
