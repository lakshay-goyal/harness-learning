---
type: failure
concepts: [shell-execution, tool-error-as-result]
harnesses: [pi]
---
**Symptom** — A bash call in a deleted/missing working directory, or with a missing shell binary, crashed the **entire agent session** with an uncaught `ENOENT` instead of returning a tool error.

**Root cause** — `spawn` errors are emitted asynchronously on the child's `error` event; without a handler they become uncaught exceptions outside the tool's promise.

**Fix · [[pi]]** — `1432fd91d` 2026-01-05 (#479, #1230): check cwd exists first with a clear error, attach `child.on("error")` → reject, fall back to `bash` on PATH (`packages/coding-agent/src/core/tools/bash.ts:97-166`).

**Lesson** — Any uncaught error inside a tool is a harness crash; pre-validate environment and convert every async failure into a tool result.

Related: [[shell-execution]] · [[tool-error-as-result]] · [[pi--shell-execution|pi]]
