---
type: failure
concepts: [context-file-hierarchy]
harnesses: [pi, opencode]
---
**Symptom** — In a git worktree nested inside its main repository, AGENTS.md was injected twice — the worktree's copy and the main repo's copy found by the ancestor walk — doubling (possibly divergent) instructions (#7221).

**Root cause** — The ancestor walk dedupes by path, but both files represent the same logical repository scope.

**Fix · [[pi]]** — `cced6a21d` 2026-07-29 (#7221) "stop loading AGENTS.md twice in nested git worktrees": `findShadowedContextFile` resolves the common git dir; when the worktree root has its own context file, the main repo's same-named copy is skipped; realpath-canonicalized (macOS `/tmp` → `/private/tmp`); bare layouts, submodules and sibling worktrees excluded (HEAD `packages/coding-agent/src/core/resource-loader.ts:205-230, 250-260`).

**Lesson** — Dedupe instruction files by logical scope (repository identity), not just file path.

Related: [[context-file-hierarchy]] · [[context-file-discovery-filesystem-edge-cases]] · [[pi--context-file-hierarchy|pi]]

**Fix · [[opencode]]** — same class (duplicate context-file injection), different trigger: reading a file pulls in nearby AGENTS.md as a `<system-reminder>`, and parallel read calls in one message each injected the same file. `558590712d` 2026-01-28 "ensure parallel tool calls dont double load AGENTS.md"; `16145af480` 2026-02-02 "prevent duplicate AGENTS.md injection when reading instruction files (#11581)": per-message `claims` set, skip files in system paths or loaded by earlier reads (`packages/opencode/src/session/instruction.ts:179-221`). See [[opencode--context-file-hierarchy]].
