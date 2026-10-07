---
type: failure
concepts: [path-normalization, shell-execution, pluggable-tool-backends]
harnesses: [pi]
---
**Symptom** — Tools ran in the wrong directory: SDK-created tools and `shellPath` resolution used the launcher's `process.cwd()` instead of the session cwd; extension-registered tools kept their creation-time cwd after the session cwd changed.

**Root cause** — cwd captured at tool construction / process level, not read per call from session state.

**Fix · [[pi]]**
- 0.26.1 — tool factories take `cwd`; 0.68.0 shell path resolved from session cwd (hashes not resolved).
- `62835ea81` 2026-09-01 (#8627) — every cwd-sensitive tool (bash, edit, find, grep, ls, read, write) uses `ctx?.cwd || cwd` (`packages/coding-agent/src/core/tools/bash.ts:271`, `path-utils.ts:48-50`).
- Durable: cwd and file identity belong to the `ExecutionEnv` (`packages/durable/src/env/index.ts:171-236`).

**Lesson** — cwd is session state, never process state; resolve it per call.

Related: [[path-normalization]] · [[shell-execution]] · [[pluggable-tool-backends]] · [[windows-process-tree-and-shells]] · [[pi--path-normalization|pi]]
