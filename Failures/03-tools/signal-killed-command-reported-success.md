---
type: failure
concepts: [shell-execution, tool-error-as-result]
harnesses: [pi]
---
**Symptom** — Shell commands killed by a signal (OOM killer, external kill) were reported to the model as **successful** with partial output; the model continued as if the build/test had passed.

**Root cause** — Node reports `code === null` (plus `signal`) for signal-terminated children; `null` was treated as non-failure.

**Fix · [[pi]]** — `a8b3dd199` 2026-09-17 (#9577): signal-killed shell → exit code `128 + signum`, null code → 1; `null` from custom `BashOperations` → throw "Command terminated without an exit code" (`packages/coding-agent/src/core/tools/bash.ts:155-158,386-388`); non-zero exit returns `isError` (`:400-407`).

**Lesson** — The model trusts exit status literally; any ambiguous termination must read as failure.

Related: [[shell-execution]] · [[tool-error-as-result]] · [[tool-result-misreports-facts]] · [[pi--shell-execution|pi]]
