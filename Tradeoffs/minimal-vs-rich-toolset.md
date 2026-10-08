---
type: tradeoff
concepts: [minimal-default-toolset, model-specific-toolset, deferred-tool-loading, replaceable-builtin-extension, file-read-tool, search-tools, shell-execution, shell-command-intent-parsing, patch-envelope-edit]
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

## Also: [[codex]] (folded from `dedicated-vs-shell-tools`)
**Axis** — does the harness give the model dedicated read / search / list tools, or a shell only (plus one edit tool) and recover intent afterwards?

| dimension | pi — dedicated tools | codex — shell + patch |
|---|---|---|
| read | `read(path, offset?, limit?)` default-on, head-truncated 2000 lines / 50 KB with "Use offset=N to continue" ([[pi--file-read-tool]]) | none for text; `cat` / `sed -n` / `rg` through `exec_command`; `view_image` for images; experimental `read_file` (indentation mode) removed `14c35a16a8` ([[codex--file-read-tool]]) |
| search / list | `grep` / `find` / `ls` on rg/fd, off by default, capped and argv-hardened ([[pi--search-tools]]) | `rg` / `rg --files` via shell (prompt `codex-rs/core/gpt_5_2_prompt.md:250`); `grep_files` / `list_dir` removed `178c3b15b4`, `70807730f5` |
| default when dedicated search off | pi tells the model "Use bash for file operations like ls, rg, find" | always the case |
| intent visible to UI / approvals | tool identity | post-hoc `parse_command` → Read / ListFiles / Search / Unknown ([[shell-command-intent-parsing]]) |
| output bounding | per tool (lines/bytes, match counts, line length) | one exec budget: 1 MiB head/tail buffer + 10k-token middle elision ([[codex--shell-execution]]) |
| safety gating | pi has no sandbox; read tools are harmless by construction | everything is a command → OS sandbox + approvals + command rules apply to reads too ([[os-level-sandbox]], [[command-rule-policy]]) |
| writes | `edit` / `write` tools | `apply_patch` only, shell writes discouraged ("Do not create or edit files with `cat` or other shell write tricks", `codex-rs/models-manager/models.json:1408`) |
| prompt cost | 4 default tool schemas (grep/find/ls opt-in) | 2 core tool schemas |
| absence notes | — | [[no-file-read-write-tools]] · [[removed-legacy-shell-tools]] |
| why chosen | small, explicit, provider-neutral; models across families | "models are trained for this, so dedicated read tools added no value" (M8; removals justified by no catalog advertising them) |

**When each wins**
- Dedicated tools: many model families of varying shell skill; harnesses without a sandbox (read tools are safe by construction); need for precise truncation/continuation protocols per tool; Windows hosts where shell quoting is hazardous ([[windows-destructive-cross-shell]]).
- Shell-only: a model family trained on shell exploration; a strong sandbox/approval layer that already mediates every command; minimizing tool-schema tokens; when UI needs can be met by parsing commands after the fact.
- Hybrid signals: pi defaults dedicated search **off** and points the model at bash; codex keeps one dedicated *write* path (`apply_patch`) even while reads go through the shell — writes are where structure pays.
