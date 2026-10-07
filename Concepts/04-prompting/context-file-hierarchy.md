---
type: concept
stage: context
tier: candidate
aliases: [AGENTS.md, CLAUDE.md, AGENTS.override.md, loadProjectContextFiles, "<project_context>", "<project_instructions>", project_doc_fallback_filenames, project_doc_max_bytes, project_root_markers, "# AGENTS.md instructions for <dir>", "<INSTRUCTIONS>", "--- project-doc ---", LoadedAgentsMd, AgentsMdManager]
harnesses: [pi, codex]
---
Instruction files (AGENTS.md / CLAUDE.md) discovered from a global dir and every ancestor down to cwd, injected into the prompt in general→specific order with explicit per-file boundaries.

## Why
- Monorepos and nested projects need layered instructions; a cwd-only lookup misses repo-root rules.
- Without dedupe/shadowing rules the same scope is applied twice (nested worktrees, [[context-file-loaded-twice-in-worktrees]]).
- Without explicit boundaries, file headings blur into the harness prompt ([[markdown-boundaries-ingested-inconsistently]]).
- Filesystem edge cases (dirs named AGENTS.md, Windows root detection, network-path probes) crash, hang or leak credentials during discovery ([[context-file-discovery-filesystem-edge-cases]]).
- Files change during long sessions; a startup snapshot goes stale ([[stale-context-files-mid-session]]).

## Design space
- Placement: as a user message (pi v1, Nov 2025; **✔ codex**, user-role `# AGENTS.md instructions for <cwd>` + `<INSTRUCTIONS>` fragment) vs **system prompt section** (**pi chose**, `b1c2c32e2`).
- Scope: cwd only (codex when no root marker found) vs **ancestors to project root marker (`.git` default, configurable `project_root_markers`, never past root)** ✔ codex vs **ancestors to filesystem root + global** (**pi**). Codex reads root markers from config layers *excluding* the project layer — a repo cannot redefine its own root.
- Names: one standard vs **ordered candidate list, first match per dir** (AGENTS.override.md, AGENTS.md, CLAUDE.md… — **pi**; AGENTS.override.md > AGENTS.md > configured fallbacks, path-syntax names rejected — ✔ codex) vs load all matches.
- Order: **root→cwd so most specific is last** (**pi**, ✔ codex; global/user instructions first) vs specific-first.
- Trust: gate repo instruction files behind project trust (pi tried for 4 days; **✔ codex** since `bd19459358`: untrusted → skip project files, keep user-level) vs **load regardless, treat as untrusted input** (**pi chose**, `5cb4f597f`).
- Fencing: markdown headings (pi until May 2026) vs **XML tags with path attribute** (**pi**) vs one fenced block for all files, single `--- project-doc ---` user→project separator, **no per-file path labels**; scoping semantics taught by the base prompt instead (✔ codex, [[no-per-file-context-labels]]).
- Budget: none (pi) vs **one shared byte budget across all files and environments (32 KiB), crossing file byte-truncated, rest dropped, truncation not told to the model** ✔ codex; host thread instructions over 10k est. tokens rejected, not truncated ✔ codex.
- Freshness: snapshot at start (pi) vs **re-read at every model-request boundary, change appended as "replaces all previously provided" notice** ✔ codex → [[world-state-diff-injection]].
- Reads through the execution environment's filesystem sandbox (blocked file fails setup) ✔ codex.
- Lazy alternative: list rule files and let the model read on demand (pi example `claude-rules.ts`); codex prompt tells the model to check for AGENTS.md when working outside cwd.

## Implementations
- [[pi--context-file-hierarchy|pi]] — global `~/.pi/agent` + ancestors, worktree shadowing, `<project_instructions path>`; not trust-gated; `--no-context-files`.
- [[codex--context-file-hierarchy|codex]] — root-marker-bounded walk, one file per dir, 32 KiB shared budget, single user-role fragment without per-file labels, trust-gated, refreshed per request with replacement notices.

## Failures
- [[markdown-boundaries-ingested-inconsistently]]
- [[context-file-loaded-twice-in-worktrees]]
- [[context-file-discovery-filesystem-edge-cases]]
- [[stale-context-files-mid-session]]
- Cross-group: [[untrusted-repo-loads-executable-config]] (07-safety) · [[permission-context-reinjected-repeatedly]] (06-caching)

## Related
[[xml-prompt-boundaries]] · [[system-prompt-override]] · [[skill-progressive-disclosure]] · [[project-trust-gate]] · [[no-prompt-injection-defense]] · [[transcript-carried-system-prompt]] · [[message-role-layering]] · [[world-state-diff-injection]] · [[cross-session-memory]]
