---
type: absence
harnesses: [codex]
---
# removed-legacy-shell-tools

Raw `shell` (argv array), Responses built-in `local_shell`, `container.exec`, `shell_command` and function-style JSON `apply_patch` were all removed — one execution surface (unified exec) and one edit surface (freeform patch) remain.

**What's missing**
- Only `exec_command` and `write_stdin` execute commands; `apply_patch` is freeform (`codex-rs/core/src/tools/spec_plan.rs:1082-1372`).

**Evidence of decision**
- `83decfa300` 2026-05-13 removed `shell`, `local_shell`, `container.exec` ("no active use").
- `e783341b70` 2026-05-08 removed function-style JSON `apply_patch`.
- `8a40095ea3` 2026-08-20 "Standardize shell execution on unified exec" removed `shell_command` ("leaving `exec_command` and `write_stdin` as the shell execution tools", 116 files, 4,704 deletions).
- History: one-shot `shell` (2025-04) → unified exec PTY sessions `c09ed74a16` 2025-09-10.

**Implication**
- Simplifies approvals and sandboxing (one exec path); legacy rollouts with old tool calls rely on replay normalization.

Related: [[shell-execution]] · [[patch-envelope-edit]] · [[no-background-bash]] · [[tool-wire-kinds]] · [[Absences]]
