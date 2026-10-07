---
type: failure
concepts: [dynamic-tool-guidelines, guideline-softening, file-read-tool]
harnesses: [pi, opencode]
---
**Symptom** — Model read files with `cat`/`sed` via bash instead of the `read` tool — losing read's truncation/paging contract, image handling and file-op tracking (compaction only tracks `read`/`write`/`edit` calls, `compaction/utils.ts:30-61`).

**Root cause** — Shell is the model's default file viewer; the guideline only said "Use read to examine files before editing", which doesn't forbid shell reads.

**Fix · [[pi]]**
- `42d7d9d9b` 2025-12-22: → "Use read to examine files before editing. You must use this tool instead of cat or sed." (silently, inside "Add before/after session events with cancellation support").
- `235b247f1` 2026-03-22: softened to "Use read to examine files instead of cat or sed." — now the read tool's own contribution (`packages/coding-agent/src/core/tools/read.ts:20-23`), emitted only when `read` is declared.

**Lesson** — Name the anti-pattern explicitly ("instead of cat or sed"); the anti-pattern naming survives, the "You must" intensity does not need to.

Related: [[dynamic-tool-guidelines]] · [[guideline-softening]] · [[file-read-tool]] · [[file-op-tracking]] · [[pi--dynamic-tool-guidelines|pi]]

**Fix · [[opencode]]** `281ce4c0c3` 2025-12-25 "prompt update to prevent searching via bash tool": "IMPORTANT: This tool is for terminal operations like git, npm, docker, etc. DO NOT use it for file operations (reading, writing, editing, searching, finding files)". HEAD shell description maps each habit to a tool: "Avoid using Bash with the `find`, `grep`, `cat`, `head`, `tail`, `sed`, `awk`, or `echo` commands…", "Read files: Use Read (NOT cat/head/tail)" (`packages/opencode/src/tool/shell/prompt.ts:100-103`); origin/v2 renders "Prefer dedicated tools over shell commands" only when `shell` is present (`origin/v2:packages/core/src/session/system-prompt.ts:11-15`). See [[opencode--tool-description-design]].
