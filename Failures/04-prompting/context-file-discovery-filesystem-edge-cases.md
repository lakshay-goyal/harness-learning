---
type: failure
concepts: [context-file-hierarchy]
harnesses: [pi]
---
**Symptom** — Context-file discovery hung or emitted spurious errors on filesystem edge cases: on Windows the ancestor walk never terminated (#6369); a *directory* named like a context file caused a yellow "Warning: Could not read /Users/…/AGENTS.md: Error: EISDIR" on every start (#7106; commit body of `58c0bc2fb`).

**Root cause** — Walk termination compared against `resolve("/")` and stepped with `resolve(currentDir, "..")` (removed lines in `2170363af`), which never matches a Windows drive root; candidate names were checked with `existsSync` only.

**Fix · [[pi]]**
- `2170363af` 2026-07-09 (#6369) "avoid Windows context file walk hang": stop when `dirname(dir) === dir` (HEAD `packages/coding-agent/src/core/resource-loader.ts:262-264`).
- `58c0bc2fb` 2026-07-25 (#7106) "exclude directories from resource loader": `statSync().isFile()` check, skip non-files (`resource-loader.ts:190-192`).

**Lesson** — Terminate path walks on the fixed point `dirname(d) === d` and verify file type, not just existence.

Related: [[context-file-hierarchy]] · [[context-file-loaded-twice-in-worktrees]] · [[pi--context-file-hierarchy|pi]]
