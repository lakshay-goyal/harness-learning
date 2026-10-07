---
type: implementation
harness: codex
concept: sandbox-escalation-retry
commit: 622e9e3696
files: [codex-rs/core/src/tools/orchestrator.rs:1, codex-rs/core/src/tools/orchestrator.rs:394, codex-rs/core/src/tools/sandboxing.rs:255, codex-rs/core/src/tools/sandboxing.rs:290, codex-rs/core/src/tools/sandboxing.rs:345, codex-rs/core/src/tools/handlers/shell_spec.rs:232, codex-rs/sandboxing/src/denial.rs:1, codex-rs/sandboxing/src/violation.rs:8]
---
[[sandbox-escalation-retry]] in [[codex]].

## Mechanism
- Orchestrator header: "Central place for approvals + sandbox selection + retry semantics … approval → select sandbox → attempt → retry with an escalated sandbox strategy on denial (no re‑approval thanks to caching)" (`codex-rs/core/src/tools/orchestrator.rs:1-7`).
- `SandboxOverride::{NoOverride, BypassSandboxFirstAttempt, EscalatedSandboxWithRestrictions}` — the last keeps the sandbox but widened ([[codex--model-requested-permissions]], `codex-rs/core/src/tools/orchestrator.rs:236-262`).
- Retry chain after `SandboxErr::Denied` (`codex-rs/core/src/tools/orchestrator.rs:394-560`):
  1. network-policy denial without approval context → return error;
  2. tool opts out (`escalate_on_failure`) → return;
  3. `wants_no_sandbox_approval(policy)` false → return — only `UnlessTrusted` and Granular with `sandbox_approval` retry: "Under `Never` or `OnRequest`, do not retry without sandbox; surface a concise sandbox denial that preserves the original output" (`codex-rs/core/src/tools/orchestrator.rs:439-442`; `codex-rs/core/src/tools/sandboxing.rs:345-352`) — except an OnRequest network prompt;
  4. denied-read policy forbids unsandboxed retry;
  5. approval prompt with `retry_reason` = `Network access to "<host>" is blocked by policy.` or text built from output (`codex-rs/core/src/tools/orchestrator.rs:474-481`);
  6. "Strict auto-review approval covers the sandboxed attempt only; retrying without the sandbox requires a fresh guardian review" (`codex-rs/core/src/tools/orchestrator.rs:484-488`).
- Model-initiated (default `on-request`): shell tool params (`codex-rs/core/src/tools/handlers/shell_spec.rs:232-276`):
  - `sandbox_permissions` enum `use_default` | `with_additional_permissions` (only when exec permission approvals enabled) | `require_escalated` — "Per-command sandbox override. Defaults to `use_default`…".
  - `justification` — "User-facing approval question for `require_escalated`; omit otherwise."
  - `prefix_rule` — "Reusable approval prefix for `cmd`, only with `sandbox_permissions: \"require_escalated\"`; for example [\"git\", \"pull\"]." → validated by [[codex--command-rule-policy]].
- Deny-read is load-bearing: `unsandboxed_execution_allowed` = no denied reads (`codex-rs/core/src/tools/sandboxing.rs:290-294`); `sandbox_permissions_preserving_denied_reads` downgrades `require_escalated` to `UseDefault` because bypassing "would drop the only mechanism that enforces them" (`codex-rs/core/src/tools/sandboxing.rs:296-310`); "a command allow rule does not authorize removing filesystem restrictions" (`codex-rs/core/src/tools/sandboxing.rs:255-263`). ExecPolicy `Allow` may yield `Skip{bypass_sandbox:true}` only otherwise (`codex-rs/core/src/tools/sandboxing.rs:265-271`).
- Denial detection `is_likely_sandbox_denied` (`codex-rs/sandboxing/src/denial.rs:1-70`): not a denial without sandbox or on exit 0; quick-reject exit codes 2/126/127 (`:25`); Linux exit `128+SIGSYS` = denial (`:30-39`); keyword scan of lower-cased stderr/stdout/aggregated: "operation not permitted", "permission denied", "read-only file system", "seccomp", "sandbox", "landlock", "failed to write file" (`:49-57`). Doc admits "a command in the user's zshrc file might hit an error".
- Violations normalized into events with output snippet ≤ 512 chars (`codex-rs/sandboxing/src/violation.rs:8-37`); remote sandbox denials reported semantically (`9f06cf1a09`).
- Escalated exec bypasses managed network proxy state (`9aaa5d9358`, [[network-policy-fail-open-paths]]).
- Per-exec variant (escalate individual `execve`s inside a script): [[codex--per-exec-interception]].

## Constants
| name | value | path:line |
|---|---|---|
| QUICK_REJECT_EXIT_CODES | 2, 126, 127 | `codex-rs/sandboxing/src/denial.rs:25` |
| SIGSYS denial | exit `128 + SIGSYS` (Linux seccomp) | `codex-rs/sandboxing/src/denial.rs:30-39` |
| denial keywords | 7 strings | `codex-rs/sandboxing/src/denial.rs:49-57` |
| violation OUTPUT_SNIPPET_MAX_CHARS | 512 | `codex-rs/sandboxing/src/violation.rs:8-37` |

## Evolution
- 2025-08-03 `e3565a3f43` "[sandbox] Filter out certain non-sandbox errors (#1804)" — quick-reject codes.
- 2025-08-05 `725dd6be6a` OnRequest: model decides when to break the sandbox.
- 2025-10-09 `ca6a0358de` keyword list ("sandbox denied error logs").
- 2025-10-20 `5e4f3bbb0b` "rework tools execution workflow (#5278)" — orchestrator.
- 2025-12-10 `e0fb3ca1db` `with_escalated_permissions` bool → `SandboxPermissions`.
- 2026-01-28 `996e09ca24` "feat(core) RequestRule (#9489)": "Instead of trying to derive the prefix_rule for a command mechanically, let's let the model decide for us."
- 2026-02-27 `6046ca19ba` prompt names DNS/registry failures as escalation triggers.
- 2026-03-08 `e6b93841c5` request_permissions tool / additional permissions.
- 2026-05-11 `6506765168` preserve managed deny-read during escalation; 2026-05-28 `6e10142199` deny-read for safe commands.
- 2026-06-22 `9f06cf1a09` remote denials semantic; 2026-07-30 `0042b00986` normalized violation events.

## Quirks
- Default mode does **not** auto-retry; reactive retry exists only for `untrusted` and granular. The model's `require_escalated` is the main path.
- Keyword list includes the bare word "sandbox" — any program printing it counts as a denial.

## Versus pi
- pi has no sandbox and so no escalation; its closest is an extension re-running a blocked call after `ui.confirm` in [[pi--tool-call-gate]].
