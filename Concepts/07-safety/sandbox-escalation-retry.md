---
type: concept
stage: permissions
tier: variant
aliases: [ToolOrchestrator, SandboxOverride, SandboxPermissions, sandbox_permissions, require_escalated, use_default, justification, escalate_on_failure, wants_no_sandbox_approval, is_likely_sandbox_denied, SandboxErr::Denied, with_escalated_permissions, escalation request]
harnesses: [codex]
---
Run a command under the sandbox first; when it is (or is about to be) blocked by the sandbox, get approval and re-run it with a widened or no sandbox — instead of pre-classifying every command as safe/unsafe. Escalation can be reactive (detect denial, retry) or model-initiated (the tool call carries an escalation request + justification).

## Why
- Pre-classification by command text needs a curated safe list that never stops needing patches and leaks (codex deleted its own, [[no-safe-command-allowlist]]).
- Sandboxed-by-default means legit work (network installs, writes outside cwd) fails; without an escalation path users switch to full access.
- Reactive detection is heuristic — "We don't have a fully deterministic way to tell if our command failed because of the sandbox" — so the model must also be able to ask up front and must be told the sandbox's failure signatures ([[sandbox-failure-misread-as-transient]], [[sandbox-network-error-not-escalated]]).
- Escalation must not silently drop restrictions that only the sandbox enforces ([[escalation-drops-deny-read]]).

## Design space
- **No sandbox → nothing to escalate** — ✔ pi.
- **Reactive retry**: sandbox denial → prompt → rerun unsandboxed. ✔ codex under `untrusted` / granular `sandbox_approval`; was the whole `on-failure` mode (removed, [[no-on-failure-approval-mode]]).
- **Model-initiated escalation in the tool schema**: `sandbox_permissions: require_escalated` + `justification` (+ reusable `prefix_rule`). ✔ codex default under `on-request`; denied commands are returned to the model, not auto-retried.
- **Scoped widening instead of full escape**: keep the sandbox but add paths / network ([[model-requested-permissions]]). ✔ codex `with_additional_permissions`.
- **Denial detection**: exit-code/keyword heuristic (✔ codex) vs kernel violation log (codex normalized violation events) vs proxy-reported semantic denial (network).
- **Re-approval on retry**: cached approval covers the retry (codex default) vs fresh review (codex strict auto-review).
- **Non-bypassable restrictions**: deny-read rules keep the sandbox even after approval (✔ codex).

## Implementations
- [[codex--sandbox-escalation-retry|codex]] — `ToolOrchestrator` approval → sandbox → attempt → escalate-on-denial; `SandboxPermissions::{UseDefault, WithAdditionalPermissions, RequireEscalated}`; denial heuristic with quick-reject exit codes and 7 keywords; deny-read downgrade.

## Failures
- [[escalation-drops-deny-read]]
- [[sandbox-failure-misread-as-transient]]
- [[sandbox-network-error-not-escalated]]
- [[model-proposed-rule-too-broad]]

## Related
[[os-level-sandbox]] · [[approval-policy-modes]] · [[model-requested-permissions]] · [[command-rule-policy]] · [[permission-state-prompt]] · [[llm-approval-reviewer]] · [[egress-policy-proxy]] · [[per-exec-interception]] · [[tool-call-gate]] · [[no-on-failure-approval-mode]] · [[no-safe-command-allowlist]]
