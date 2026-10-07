---
type: concept
stage: tools
tier: candidate
aliases: [resolveReadPath, resolveToCwd, resolvePath, path-input-normalization, ctx.cwd, LocationMutation.resolve]
harnesses: [pi, opencode]
---
Normalize path quirks coming from models and users (leading `@`, `~`, unicode spaces, `file://`, MSYS/WSL drive forms, macOS screenshot filenames with invisible characters) and resolve them against the session's current cwd before any file tool touches the filesystem.

## Why
- macOS screenshot names contain U+202F before AM/PM, NFD forms, curly apostrophes; typed paths don't match ([[read-path-unicode-variants]]).
- CLI habits (`@file`) and Git-Bash drive forms (`/c/...`) leak into tool arguments.
- Resolving against process cwd instead of session cwd sends tools to the wrong tree after a cwd change ([[tools-ignore-session-cwd]]).

## Design space
- Pass-through (OS resolves) vs **normalize then resolve** (pi).
- Try variants on failure (pi read: screenshot variants only after the plain path fails) vs normalize eagerly.
- Cwd confinement / allowlist (pi rejected: [[no-cwd-confinement]]; early pi briefly validated "cwd + ancestors", see [[read-path-traversal]]) vs none.
- Symlink-escape checks (absent in pi).
- Session cwd from per-call context (pi `ctx.cwd`) vs captured at tool construction.
- Canonical (realpath) containment with typed escape reasons (opencode v2) vs lexical containment (opencode legacy) → [[workspace-boundary-check]].

## Implementations
- [[pi--path-normalization|pi]] — `resolveToCwd` → `resolvePath` (unicode spaces, `@`, MSYS/WSL/Cygwin drives, `~`, `file://`), read-only screenshot variants, `ctx.cwd || cwd`.
- [[opencode--path-normalization|opencode]] — legacy: relative paths joined to the instance dir, lexical containment → `external_directory` ask; v2: realpath containment with typed escape reasons.

## Failures
- [[read-path-unicode-variants]]
- [[tools-ignore-session-cwd]]
- [[find-glob-semantics-mismatch]]
- [[windows-process-tree-and-shells]]

## Tradeoffs
- [[cwd-confinement-vs-none]]

## Related
[[file-read-tool]] · [[search-tools]] · [[shell-execution]] · [[pluggable-tool-backends]]
