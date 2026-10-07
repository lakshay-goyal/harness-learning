---
type: tradeoff
concepts: [minimal-default-toolset, file-read-tool, search-tools, shell-execution, shell-command-intent-parsing, patch-envelope-edit]
---
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
