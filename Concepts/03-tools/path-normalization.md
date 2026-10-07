---
type: concept
stage: tools
tier: candidate
aliases: [resolveReadPath, resolveToCwd, resolvePath, path-input-normalization, ctx.cwd, PathUri, resolve_path]
harnesses: [pi, codex]
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
- Environment-native path type resolved against the *selected environment's* cwd, never projected onto the host (✔ codex `PathUri`) → [[pluggable-tool-backends]].
- De-alias paths inside one batch edit (`a` vs `./a` rejected) (✔ codex `a1c88e865d`).
- Canonical-path evaluation for sandbox roots (✔ codex) → [[symlinked-roots-escape-sandbox-policy]].

## Implementations
- [[pi--path-normalization|pi]] — `resolveToCwd` → `resolvePath` (unicode spaces, `@`, MSYS/WSL/Cygwin drives, `~`, `file://`), read-only screenshot variants, `ctx.cwd || cwd`.
- [[codex--path-normalization|codex]] — partial: `PathUri` joins against environment cwd (+ `cd dir &&` workdir for patches); duplicate resolved paths rejected; no `@`/`~`/unicode quirk layer found.

## Failures
- [[read-path-unicode-variants]]
- [[tools-ignore-session-cwd]]
- [[find-glob-semantics-mismatch]]
- [[windows-process-tree-and-shells]]
- [[duplicate-path-ops-in-one-patch]]
- [[apply-patch-path-and-permission-hazards]]
- (07) [[symlinked-roots-escape-sandbox-policy]]
- [[windows-backslash-cwd-copied-into-shell]] (04-prompting) — On Windows, the model copied the cwd from the system prompt (C:\Users\…) into bash commands, where…

## Related
[[file-read-tool]] · [[search-tools]] · [[shell-execution]] · [[pluggable-tool-backends]] · [[patch-envelope-edit]]
