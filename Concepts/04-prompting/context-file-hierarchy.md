---
type: concept
stage: context
tier: candidate
aliases: [AGENTS.md, CLAUDE.md, AGENTS.override.md, loadProjectContextFiles, "<project_context>", "<project_instructions>", Instruction.resolve, "Instructions from: <path>", CONTEXT.md, OPENCODE_DISABLE_CLAUDE_CODE_PROMPT]
harnesses: [pi, opencode]
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
- Lazy nested discovery: attach instruction files from directories the model reads into, once per message (opencode).
- Filename precedence across the whole walk, never mixing AGENTS.md and CLAUDE.md (opencode).
- Remote instruction URLs with a timeout (opencode, 5 s, failures silent).
- Instruction changes delivered as a transcript update ("These instructions replace all previously loaded ambient instructions.") (opencode v2) → [[transcript-carried-system-prompt]].

## Implementations
- [[pi--context-file-hierarchy|pi]] — global `~/.pi/agent` + ancestors, worktree shadowing, `<project_instructions path>`; not trust-gated; `--no-context-files`.
- [[opencode--context-file-hierarchy|opencode]] — global first hit (`~/.config/opencode/AGENTS.md` or `~/.claude/CLAUDE.md`); project: first filename with any match, all its ancestors cwd→worktree; config globs/URLs; nested files attached to read results as `<system-reminder>`.

## Failures
- [[markdown-boundaries-ingested-inconsistently]]
- [[context-file-loaded-twice-in-worktrees]]
- [[context-file-discovery-filesystem-edge-cases]]
- [[generic-context-file-bloat]]

## Related
[[xml-prompt-boundaries]] · [[system-prompt-override]] · [[skill-progressive-disclosure]] · [[project-trust-gate]] · [[no-prompt-injection-defense]] · [[transcript-carried-system-prompt]]
