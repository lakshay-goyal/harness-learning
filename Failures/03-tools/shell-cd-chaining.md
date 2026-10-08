---
type: failure
concepts: [shell-execution, tool-description-design]
harnesses: [opencode]
---
**Symptom** — The model prefixed nearly every shell command with `cd <dir> &&` ("cd spam"), even for the project root.

**Root cause** — Habit from shell-centric training; the tool gave no parameter for the working directory and did not say where commands run.

**Fix · [[opencode]]**
- `75a4dcbce8` 2025-12-06 `workdir` parameter; "Prefer using `workdir` over `cd <dir> &&`".
- `4e92f54415` 2025-12-11 "try to prevent the cd spam": "All commands run in ${directory} by default".
- `160c8ab7cc` 2025-12-26 "AVOID using `cd <directory> && <command>` patterns - use `workdir` instead" + good/bad example.
- HEAD: `packages/opencode/src/tool/shell/prompt.ts:112,261` (directory text made static for caching, `38014fe448`).

**Lesson** — Give a parameter that replaces the habit, then name and forbid the habit.

Related: [[shell-execution]] · [[tool-description-design]] · [[opencode--tool-description-design|opencode]]
