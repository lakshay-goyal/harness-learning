---
type: absence
harnesses: [pi, opencode, codex]
---
# no-codebase-index

No embeddings, RAG, vector store, repo map or tree-sitter symbol index.

**What's missing**
- Nothing pre-indexes or summarizes the repository for the model. Exploration is on-demand via bash (`rg`, `find`, `ls`) or the opt-in grep/find/ls tools ([[search-tools]]).
- `git grep -i -E 'embedding|vector store|repo.?map|tree-sitter'` over `packages/coding-agent/src` and `packages/agent/src` has two hits, neither an index:
  - the BM25 tool-ranker comment, "BM25 today; a hybrid ranker with embeddings can replace it" (`packages/coding-agent/src/extensions/tool-search/tool.ts:34`), which ranks *tools*, not code;
  - "embedding the agent in other applications" (`src/modes/rpc/rpc-mode.ts:4`).
- `durable/src` also has no hits.

**Evidence of decision**
- Absence by omission. There is no explicit statement and no commit adding or removing a repo map.
- `search-index` commits (`b92e5e861` 2026-07-27 "search index factory", `b75be04d9` 2026-08-11 "refactor: search (#7797)") index *sessions* (sqlite/jsonl), not code.
- Design signal: grep/find/ls exist but are **off by default** (`packages/coding-agent/docs/settings.md:40,44`; default `read, bash, edit, write`, `settings-manager.ts:215`). The agent is expected to explore with bash/rg.
- The prompt rule "Prefer grep/find/ls tools over bash for file exploration (faster, respects .gitignore)" was removed in `1ab289980` (2026-05-28, #5132) because it preferred tools that might not be active. The guideline was changed to "ls, rg, find" in `b846a4bfc`.

**Opt-in replacement**
- Context files: AGENTS.md/CLAUDE.md hierarchy ([[context-file-hierarchy]]). These are human-written maps, not computed ones.
- `examples/extensions/claude-rules.ts` lists `.claude/rules/` files in the system prompt for on-demand reading ([[skill-progressive-disclosure]]).
- MCP servers offering code search (since `8562bcf66`). The `tool_search` BM25 ranker is designed so embeddings could replace it (`tool-search/tool.ts:2,34`) → [[deferred-tool-loading]].
- No example extension provides embeddings/RAG.

**History**
- Never present.

**Implication**
- Every session starts cold. Discovery cost is paid in tool calls and tokens each time, which is offset by prefix caching and compaction.
- No index means no staleness, no background indexing process, and no extra security surface (an index can leak files to an embedding API).

**codex** — *absent too*. No embeddings, vector store or repo map. `tree-sitter`, `tree-sitter-bash`, `tree-sitter-powershell` are workspace deps (`codex-rs/Cargo.toml:534-536`) used only by `codex-rs/apply-patch` and `codex-rs/shell-command` (command parsing/safety), not indexing. BM25 exists only for *tool* search (`codex-rs/core/src/tools/handlers/tool_search.rs:14-16`) and a shadow-metrics lexical *skill* selector (`c100109280` 2026-07-13). `codex-rs/file-search` (born `296996d74e` 2025-06-25) is fuzzy filename search for the UI @-mention picker. The model explores with `rg` / `rg --files` (prompt guidance, e.g. `codex-rs/core/gpt_5_2_prompt.md:250`).
**opencode** ([[opencode]], `ecc4916b5a`): also absent.
- No embeddings or vector index in `packages/opencode/src` or `packages/core/src` (grep `embedding|vector`; the only "file search" is Copilot's hosted Responses tool, `packages/core/src/github-copilot/responses/tool/file-search.ts`).
- Substitutes: ripgrep-backed `glob`/`grep` ([[search-tools]]), the `explore` subagent (read-only, own prompt; `packages/opencode/src/agent/agent.ts:196-218`), experimental LSP `workspaceSymbol` ([[lsp-diagnostics-feedback]]), and config `references` to external repos ([[project-references]]).
- The repo-research `scout` agent with `repo_clone`/`repo_overview` lived 2026-05-09 → 2026-06-02 (`40d5ea1cf1` → `a639fe7a08`) → [[removed-builtin-tools]].

Related: [[search-tools]] · [[minimal-default-toolset]] · [[context-file-hierarchy]] · [[skill-progressive-disclosure]] · [[deferred-tool-loading]] · [[no-lsp]] · [[Absences]] · [[shell-command-intent-parsing]] · [[minimal-vs-rich-toolset]]
Related: [[search-tools]] · [[minimal-default-toolset]] · [[context-file-hierarchy]] · [[skill-progressive-disclosure]] · [[deferred-tool-loading]] · [[no-lsp]] · [[opencode]] · [[Absences]]
