---
type: concept
stage: permissions
tier: variant
aliases: [request_permissions, with_additional_permissions, additional_permissions, AdditionalPermissionProfile, RequestPermissionProfile, PermissionGrantScope, EscalatedSandboxWithRestrictions, scoped sandbox widening]
harnesses: [codex]
---
A dedicated tool (or per-call parameter) lets the model ask for a *scoped* widening of its sandbox profile — specific paths readable/writable, network on — for the turn or the session, instead of all-or-nothing unsandboxed escalation.

## Why
- Binary escalation ("run without sandbox") over-grants: to write one file outside cwd the model drops every restriction including network and deny-reads ([[escalation-drops-deny-read]]).
- Users approve concrete capabilities ("write `~/proj2`", "network") more confidently than opaque commands; grants can be reused for later commands in the turn/session without re-prompting.

## Design space
- **None / full escape only** — pi (no sandbox); codex before `e6b93841c5`.
- **Per-command widening** (`sandbox_permissions: with_additional_permissions` + `additional_permissions{network.enabled, file_system.read/write}`) — ✔ codex.
- **Standalone request tool** returning a granted subset with scope `Turn | Session` — ✔ codex `request_permissions`.
- **Client may grant a subset** (✔ codex) vs all-or-nothing.
- **Gating by policy**: Granular `request_permissions=false` auto-rejects (✔ codex); reviewer may handle the request (guardian kind `RequestPermissions`).

## Implementations
- [[codex--model-requested-permissions|codex]] — `request_permissions` tool + `with_additional_permissions`; `EscalatedSandboxWithRestrictions`; grant scope Turn/Session; prompt prefers it over `require_escalated`.

## Failures
- [[escalation-drops-deny-read]]

## Related
[[sandbox-escalation-retry]] · [[os-level-sandbox]] · [[approval-policy-modes]] · [[permission-state-prompt]] · [[llm-approval-reviewer]] · [[ask-user-tool]]
