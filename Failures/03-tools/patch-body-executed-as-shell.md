---
type: failure
concepts: [shell-execution, patch-envelope-edit]
harnesses: [codex]
---
**Symptom** — "sometimes the model forgets to actually invoke `apply_patch` and puts a patch as the script body. trying to execute this as bash sometimes creates files named `,` or `{`" (`5f6e95b592` body).

**Root cause** — The shell tool executed whatever it got; a patch body is valid-looking shell to bash (redirections, braces), so a mis-routed edit produced junk side effects instead of an error.

**Fix · [[codex]]**
- `5f6e95b592` 2025-09-12 "if a command parses as a patch, do not attempt to run it (#3382)" — a raw patch passed as the whole command, or as the body of a `bash -lc` script, returns `ApplyPatchError::ImplicitInvocation`: `patch detected without explicit call to apply_patch. Rerun as ["apply_patch", "<patch>"]` (`codex-rs/apply-patch/src/invocation.rs:141-157`; `codex-rs/apply-patch/src/lib.rs:86-90`; tests `invocation.rs:521-538`).
- Correct heredoc invocations through `exec_command` are routed to the patch path instead of executing (`codex-rs/core/src/tools/handlers/unified_exec/exec_command.rs:372`).

**Lesson** — Sniff shell input for your edit DSL before executing; a mis-routed edit must fail loudly with the corrected invocation, never run.

Related: [[shell-execution]] · [[patch-envelope-edit]] · [[tool-argument-repair]] · [[codex--shell-execution|codex shell]] · [[codex--patch-envelope-edit|codex patch]]
