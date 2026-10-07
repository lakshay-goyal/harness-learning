---
type: implementation
harness: codex
concept: approval-key-canonicalization
commit: 622e9e3696
files: [codex-rs/core/src/command_canonicalization.rs:5, codex-rs/core/src/tools/approvals.rs:161, codex-rs/core/src/tools/approvals.rs:239]
---
[[approval-key-canonicalization]] in [[codex]].

## Mechanism
- `canonicalize_command_for_approval` (`codex-rs/core/src/command_canonicalization.rs:5-37`): a single plain command inside `bash -lc` → that argv; otherwise `["__codex_shell_script__", <mode>, <exact script>]`; PowerShell → `["__codex_powershell_script__", <script>]`. Doc: stable "across wrapper-path differences (for example `/bin/bash -lc` vs `bash -lc`)" (`:8-13`).
- Used for the session approval cache `ApprovalCacheKey::{ExecCommand, ApplyPatch}` (`codex-rs/core/src/tools/approvals.rs:161`, `:239-280`).
- Parallel exec approvals keyed by request id, not turn (`c4b771a16f`); patch approvals and elicitations carry `call_id` (`084236f717`, `6a0f709cff`).

## Evolution
- 2025-07-23 `084236f717` call_id on patch approvals/elicitations; 2025-08-14 `6a0f709cff` call_id on MCP approvals.
- 2026-02-10 `62d0f302fd` "canonicalize wrapper approvals and support heredoc prefix … (#10941)"; same day `c4b771a16f` approve parallel exec on request id.

## Versus pi
- No approval cache in pi (no approvals).
