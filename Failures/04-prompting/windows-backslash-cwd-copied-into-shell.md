---
type: failure
concepts: [minimal-system-prompt, path-normalization]
harnesses: [pi]
---
**Symptom** — On Windows, the model copied the cwd from the system prompt (`C:\Users\…`) into bash commands, where backslashes are escapes → broken paths and failing commands (#2080).

**Root cause** — The prompt rendered `process.cwd()` verbatim; the bash tool runs a POSIX-style shell (Git Bash/MSYS) on Windows.

**Fix · [[pi]]** — `671798d67` 2026-03-14 (#2080) "normalize prompt cwd for bash-safe windows paths": backslashes → "/" (HEAD `packages/coding-agent/src/core/system-prompt.ts:183` `cwd.replace(/\\/g, "/")`; CHANGELOG `:3091` "Windows backslashes breaking bash tool execution").

**Lesson** — Every path you put in a prompt is a literal the model will paste into tools; render it in the syntax of the tool that will consume it.

Related: [[minimal-system-prompt]] · [[path-normalization]] · [[shell-execution]] · [[pi--minimal-system-prompt|pi]]
