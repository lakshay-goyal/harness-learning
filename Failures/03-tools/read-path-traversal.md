---
type: failure
concepts: [path-normalization, tool-only-isolation]
harnesses: [pi]
---

**Symptom** — `read` tool could read files anywhere on disk via `../` or absolute paths. Reported as a "path traversal vulnerability" in 0.11.7 (2025-12-01).

**Root cause** — File tools resolved model-supplied paths against cwd with no confinement.

**Fix · [[pi]]** — 0.11.7: "Paths are now validated to prevent reading outside the working directory or its parents. The `read` tool can read from `cwd`, its ancestors (for config files), and all descendants. Symlinks are resolved before validation." (`packages/coding-agent/CHANGELOG.md:6023`, release heading `:6019`). Fix commit hash: unverified (`git log -S` finds no introducing commit). **Later reversed:** at HEAD `resolveToCwd` performs no confinement, and `docs/security.md` delegates isolation to OS/containers → [[no-cwd-confinement]]; see [[pi--path-normalization|pi]].

**Lesson** — Path confinement in a full-shell agent is cosmetic (bash can `cat` anything). Pi first patched it, then dropped it for honest "no sandbox" docs plus opt-in isolation ([[no-sandbox]], [[tool-only-isolation]]).

Related: [[path-normalization]] · [[tool-call-gate]] · [[no-permission-prompts]]
