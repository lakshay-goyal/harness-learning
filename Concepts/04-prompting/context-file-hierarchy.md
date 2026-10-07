---
type: concept
stage: context
tier: candidate
aliases: [AGENTS.md, CLAUDE.md, AGENTS.override.md, loadProjectContextFiles, "<project_context>", "<project_instructions>"]
harnesses: [pi]
---
Instruction files (AGENTS.md / CLAUDE.md) discovered from a global dir and every ancestor down to cwd, injected into the prompt in general→specific order with explicit per-file boundaries.

## Why
- Monorepos and nested projects need layered instructions; a cwd-only lookup misses repo-root rules.
- Without dedupe/shadowing rules the same scope is applied twice (nested worktrees, [[context-file-loaded-twice-in-worktrees]]).
- Without explicit boundaries, file headings blur into the harness prompt ([[markdown-boundaries-ingested-inconsistently]]).
- Filesystem edge cases (dirs named AGENTS.md, Windows root detection) crash or hang discovery ([[context-file-discovery-filesystem-edge-cases]]).

## Design space
- Placement: as a user message (pi v1, Nov 2025) vs **system prompt section** (**pi chose**, `b1c2c32e2`).
- Scope: cwd only vs ancestors to git root vs **ancestors to filesystem root + global** (**pi**).
- Names: one standard vs **ordered candidate list, first match per dir** (AGENTS.override.md, AGENTS.md, CLAUDE.md… — **pi**) vs load all matches.
- Order: **root→cwd so most specific is last** (**pi**) vs specific-first.
- Trust: gate repo instruction files behind project trust (pi tried for 4 days) vs **load regardless, treat as untrusted input** (**pi chose**, `5cb4f597f`).
- Fencing: markdown headings (pi until May 2026) vs **XML tags with path attribute** (**pi**).
- Lazy alternative: list rule files and let the model read on demand (pi example `claude-rules.ts`).

## Implementations
- [[pi--context-file-hierarchy|pi]] — global `~/.pi/agent` + ancestors, worktree shadowing, `<project_instructions path>`; not trust-gated; `--no-context-files`.

## Failures
- [[markdown-boundaries-ingested-inconsistently]]
- [[context-file-loaded-twice-in-worktrees]]
- [[context-file-discovery-filesystem-edge-cases]]

## Related
[[xml-prompt-boundaries]] · [[system-prompt-override]] · [[skill-progressive-disclosure]] · [[project-trust-gate]] · [[no-prompt-injection-defense]] · [[transcript-carried-system-prompt]]
