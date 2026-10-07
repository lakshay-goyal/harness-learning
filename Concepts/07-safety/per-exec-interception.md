---
type: concept
stage: permissions
tier: candidate
aliases: [codex-shell-escalation, codex-execve-wrapper, EXEC_WRAPPER, CODEX_ESCALATE_SOCKET, EscalationDecision, patched zsh, patched bash, zsh exec bridge, ApprovalAction::Execve, codex-shell-tool-mcp, exec-server]
harnesses: [codex]
---
Patch the shell so every `execve` inside a sandboxed script is routed to the harness, which decides per sub-command Run (inside the sandbox) / Escalate (run outside with forwarded fds) / Deny — policy at exec granularity instead of whole-script granularity.

## Why
- Whole-command policy is coarse: approving `make test` escalates everything the Makefile runs; denying it blocks the one sub-step that needed network.
- Parsing the script text ([[shell-command-intent-parsing]], [[command-rule-policy]]) can't see what `npm run x` or a function call actually executes; the kernel exec boundary can.
- Lets rules like `prefix_rule(["git","push"], "prompt")` fire even when `git push` is buried in a script.

## Design space
- **Whole-command decision** (approval and sandbox per tool call) — pi (no decision at all); codex default.
- **Exec interception via patched shell + wrapper binary** (✔ codex: zsh `EXEC_WRAPPER`, earlier bash patch) vs. `LD_PRELOAD`/seccomp-notify/ptrace interception (not used by codex).
- **Decisions**: run sandboxed / escalate unsandboxed with fd passing / deny with exit 1.
- **Approval routing**: same approval seam + reviewer kind `Execve` (✔ codex).

## Implementations
- [[codex--per-exec-interception|codex]] — `codex-execve-wrapper` + `CODEX_ESCALATE_SOCKET` protocol; patched zsh (`codex-zsh-vX.Y.Z` releases); `ApprovalAction::Execve`; Unix only.

## Failures
- none recorded

## Related
[[sandbox-escalation-retry]] · [[command-rule-policy]] · [[os-level-sandbox]] · [[shell-execution]] · [[llm-approval-reviewer]] · [[tool-call-gate]]
