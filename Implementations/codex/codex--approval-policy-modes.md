---
type: implementation
harness: codex
concept: approval-policy-modes
commit: 622e9e3696
files: [codex-rs/protocol/src/protocol.rs:1028, codex-rs/protocol/src/protocol.rs:4181, codex-rs/core/src/tools/sandboxing.rs:196, codex-rs/core/src/tools/approvals.rs:161, codex-rs/core/src/tools/approvals.rs:505, codex-rs/core/src/tools/approvals.rs:712, codex-rs/core/src/config/mod.rs:3753, codex-rs/utils/cli/src/shared_options.rs:40]
---
[[approval-policy-modes]] in [[codex]].

## Mechanism
- `AskForApproval` (`codex-rs/protocol/src/protocol.rs:1028-1052`):
  - `UnlessTrusted` (serialized `untrusted`) — "Internal policy for projects marked untrusted. Commands require approval unless an explicit exec policy rule allows them."
  - `OnRequest` — `#[default]`, "The model decides when to ask the user for approval"; `on-failure` survives only as `#[serde(alias)]` (`codex-rs/protocol/src/protocol.rs:1036`).
  - `Granular(GranularApprovalConfig)` — flags `sandbox_approval` (incl. inline `with_additional_permissions` / `require_escalated`), `rules` (execpolicy `prompt` rules), `skill_approval`, `request_permissions`, `mcp_elicitations`; `false` = auto-reject instead of prompting (`codex-rs/protocol/src/protocol.rs:1039-1067`).
  - `Never` — "Failures are immediately returned to the model, and never escalated to the user for approval."
- Explicit `approval_policy = "untrusted"` in config is now an error (`UnsupportedUntrustedApprovalPolicyError`, `codex-rs/core/src/config/mod.rs:3753-3758`); `UnlessTrusted` is only reached via a project explicitly marked untrusted; trusted → `OnRequest` (`codex-rs/core/src/config/mod.rs:3759-3770`). See [[codex--project-trust-gate]].
- Default exec requirement `default_exec_approval_requirement` (`codex-rs/core/src/tools/sandboxing.rs:196-230`): Never → `Skip`; OnRequest/Granular → `NeedsApproval` only if the filesystem policy is `Restricted`; UnlessTrusted → always; Granular with `sandbox_approval=false` → `Forbidden{"approval policy disallowed sandbox approval prompt"}`. Then refined per command by [[command-rule-policy]] and [[dangerous-command-heuristics]].
- `ReviewDecision` (`codex-rs/protocol/src/protocol.rs:4181-4214`): `Approved` · `ApprovedExecpolicyAmendment` (persist a prefix rule) · `ApprovedForSession` (session cache) · `ApprovedMcpPolicyAmendment` (cross-session MCP allow) · `NetworkPolicyAmendment` (persist host allow/deny) · `Denied{rejection}` (turn continues) · `TimedOut` (auto-review timeout) · `Abort` (stop until next user message). `Default` = `Denied` → fail-closed.
- Who answers — `request_approval` precedence (`codex-rs/core/src/tools/approvals.rs:505-525`, `:554-569`): 1) PermissionRequest hooks (allow/deny, or no decision → continue; a failing hook yields no decision, `codex-rs/hooks/src/events/permission_request.rs:197-205`), 2) guardian if strict auto-review or reviewer enabled ([[codex--llm-approval-reviewer]]), 3) user.
- Session cache: `ApprovalCacheKey::{ExecCommand, ApplyPatch}` (`codex-rs/core/src/tools/approvals.rs:161`, `:239-280`); `with_cached_approval` (`codex-rs/core/src/tools/approvals.rs:712`); keys canonicalized ([[codex--approval-key-canonicalization]]). Exec approvals keyed by request id since `c4b771a16f`.
- apply_patch path (`assess_patch_safety`, `codex-rs/core/src/safety.rs:67-125`): empty patch rejected; UnlessTrusted → ask (with a TODO admitting doubt, `:83-88`); patch confined to writable paths + sandbox available → auto-approve but still run sandboxed because "paths in the patch are hard links to files outside the writable roots" (`:101-112`); else reject under Never / granular-no-sandbox, else ask.
- CLI (`codex-rs/utils/cli/src/shared_options.rs:40-63`): `--sandbox/-s`; `--approve-for-me` (alias `not-so-yolo`) = route approvals to auto-review with workspace-write, conflicts with `--sandbox` and bypass; `--dangerously-bypass-approvals-and-sandbox` (alias `yolo`); `--dangerously-bypass-hook-trust` ("DANGEROUS. Intended only for automation that already vets hook sources").
- Admin constraints: requirements allow-lists `allowed_approval_policies`, `allowed_sandbox_modes`, `allowed_permission_profiles` (`codex-rs/config/src/config_requirements.rs:1030-1071`); delegates pinned via `Constrained::allow_only(AskForApproval::Never)` (`codex-rs/core/src/codex_delegate.rs:73`).
- Model-facing text per mode: [[codex--permission-state-prompt]].

## Constants
| name | value | path:line |
|---|---|---|
| default approval policy | `OnRequest` | `codex-rs/protocol/src/protocol.rs:1035-1038` |
| default ReviewDecision | `Denied` | `codex-rs/protocol/src/protocol.rs:4209-4214` |
| PROMPT_CONFLICT_REASON | "approval required by policy, but AskForApproval is set to Never" | `codex-rs/core/src/exec_policy.rs:48-53` |

## Evolution
- 2025-04 TS CLI modes suggest / auto-edit / full-auto → execpolicy crate `58f0e5ab74` 2025-04-24 (M8 timeline).
- 2025-06-24/25 `86d5a9d80d` / `72082164c1` rename unless-allow-listed → `untrusted`.
- 2025-06-25 `50924101d2` `--dangerously-bypass-approvals-and-sandbox`.
- 2025-08-05 `725dd6be6a` "[approval_policy] Add OnRequest approval_policy (#1865)": "Let's try something new: tell the model about the sandbox, and let it decide when it will need to break the sandbox".
- 2025-09-26 `55801700de` reject dangerous commands under `Never`.
- 2026-02-10 `c4b771a16f` approve parallel exec on request id.
- 2026-02-12 `4668feb43a` "Deprecate approval_policy: on-failure (#11631)" — "it performs worse than `on-request`, and we're focusing on making fewer sandbox configurations perform much better".
- 2026-04-28 `3d10ba9f36` deprecate `--full-auto`; 2026-07-30 `1c5f336c40` removed from `codex exec`; 2026-07-31 `b7a6106608` `--approve-for-me`.
- 2026-06-23 `2cf2a6a844` "rm AskForApproval::OnFailure (#28418)".
- 2026-08-19 `942af8447b` "Retire the untrusted approval policy (#39630)": user-facing mode removed, known-safe allowlist deleted ([[no-safe-command-allowlist]]); untrusted projects prompt for every command not allowed by a rule.

## Quirks
- `untrusted` is now an *internal* mode chosen by trust, not by config.
- `Never` still forbids dangerous commands and rule `prompt` decisions (conflict → forbidden with PROMPT_CONFLICT_REASON).

## Versus pi
- pi: no approval layer ([[no-permission-prompts]]); [[pi--tool-call-gate]] is the seam for opt-in confirmations. Codex bakes the matrix into core and the model prompt.
