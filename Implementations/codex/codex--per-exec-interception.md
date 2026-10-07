---
type: implementation
harness: codex
concept: per-exec-interception
commit: 622e9e3696
files: [codex-rs/shell-escalation/README.md:6, codex-rs/shell-escalation/src/unix/escalate_protocol.rs:11, codex-rs/core/src/tools/approvals.rs:498, codex-rs/arg0/src/lib.rs:24]
---
[[per-exec-interception]] in [[codex]].

## Mechanism
- `codex-execve-wrapper` "receives the arguments to an intercepted `execve(2)` call and delegates the decision to the shell-escalation protocol over a shared file descriptor (specified by the `CODEX_ESCALATE_SOCKET` environment variable)" (`codex-rs/shell-escalation/README.md:6-8`). Replies (`:10-16`; `codex-rs/shell-escalation/src/unix/escalate_protocol.rs:11-75`):
  - `Run` — wrapper `execve`s itself into the original command inside the sandboxed shell;
  - `Escalate` — forward the process's fds so the command runs "faithfully outside the sandbox"; exit code forwarded back;
  - `Deny` — print to stderr, exit 1.
- Requires a patched shell: zsh `Src/exec.c` patch adding `EXEC_WRAPPER`, applied to zsh commit `77045ef899e53b9598bebc5a41db93a548a40ca6`; release artifacts on `codex-zsh-vX.Y.Z` tags, consumed via DotSlash manifests (`codex-rs/shell-escalation/README.md:18-34`). Earlier: bash patch `shell-tool-mcp/patches/bash-exec-wrapper.patch` built for many distros (`d363a0968e`).
- Approvals: `ApprovalAction::Execve` with `approval_id` through the normal approval seam (`codex-rs/core/src/tools/approvals.rs:498-500`); guardian request kind `Execve` ([[codex--llm-approval-reviewer]]).
- Dispatch: arg0 alias `codex-execve-wrapper` in the multitool binary (`codex-rs/arg0/src/lib.rs:24`). Unix only.
- Launch-context env (session ids, auth tokens) scrubbed from children (`c4513cb982`; `codex-rs/protocol/src/shell_environment.rs:13-21`).

## Evolution
- 2025-11-18 `c1391b9f94` "exec-server (#6630)".
- 2025-11-21 `d363a0968e` "codex-shell-tool-mcp (#7005)" — patched bash as an MCP shell tool.
- 2026-02-17 `edacbf7b6e` "zsh exec bridge (#12052)".
- 2026-02-23 `5221575f23` / `38f84b6b29` / `af215eb390` extracted into `codex-shell-escalation`, decoupled from core.
- 2026-08-08 `c4513cb982` "Prevent launch context from reaching child processes (#37607)".

## Quirks
- Shipping a patched shell is part of the security boundary; the zsh commit is pinned.
