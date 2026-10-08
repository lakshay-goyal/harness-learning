---
type: failure
concepts: [path-normalization, tool-only-isolation]
harnesses: [pi, opencode]
---

**Symptom** — `read` tool could read files anywhere on disk via `../` or absolute paths. Reported as a "path traversal vulnerability" in 0.11.7 (2025-12-01).

**Root cause** — File tools resolved model-supplied paths against cwd with no confinement.

**Fix · [[pi]]** — 0.11.7: "Paths are now validated to prevent reading outside the working directory or its parents. The `read` tool can read from `cwd`, its ancestors (for config files), and all descendants. Symlinks are resolved before validation." (`packages/coding-agent/CHANGELOG.md:6023`, release heading `:6019`). Fix commit hash: unverified (`git log -S` finds no introducing commit). **Later reversed:** at HEAD `resolveToCwd` performs no confinement, and `docs/security.md` delegates isolation to OS/containers → [[no-cwd-confinement]]; see [[pi--path-normalization|pi]].

**Fix · [[opencode]]** (prevention, not a dated fix) — paths outside the project directory or worktree raise an `external_directory` permission ask keyed by `<parent dir>/*` rather than a hard block (`packages/opencode/src/tool/external-directory.ts:15-44`; `packages/opencode/src/project/instance-context.ts:18-24`; non-git worktree `/` ignored). Legacy containment is lexical: `FSUtil.contains` is `path.relative` with no realpath (`packages/core/src/fs-util.ts:270-273`), so an in-project symlink pointing outside counts as inside (observed; exploitability unverified). v2 read and mutation tools canonicalize with `fs.realPath` and require `external_directory` approval for external absolute paths (`packages/core/src/location-mutation.ts:84-103`; `packages/core/src/tool/read.ts:42`, `:64`; `specs/v2/schema-changelog.md:253-269`).

**Lesson** — Path confinement in a full-shell agent is cosmetic (bash can `cat` anything). Pi first patched it, then dropped it for honest "no sandbox" docs plus opt-in isolation ([[no-sandbox]], [[tool-only-isolation]]).

Related: [[path-normalization]] · [[tool-call-gate]] · [[no-permission-prompts]]
