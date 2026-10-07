---
type: concept
stage: permissions
tier: candidate
aliases: [AskForApproval, approval_policy, ask_for_approval, untrusted, unless-trusted, on-request, on-failure, never, granular, GranularApprovalConfig, ExecApprovalRequirement, ReviewDecision, ApprovalCacheKey, --ask-for-approval, --approve-for-me, --dangerously-bypass-approvals-and-sandbox, suggest / auto-edit / full-auto]
harnesses: [codex]
---
A user/admin-selected policy that decides, per action category, whether the harness asks a human (or reviewer), auto-approves, or auto-rejects; orthogonal to the sandbox policy, and the two combine into the effective permission.

## Why
- Without a policy knob there are only two products: YOLO (pi [[no-permission-prompts]]) or prompt-on-everything (approval fatigue → users rubber-stamp).
- Separating "who decides" (approval) from "what the kernel allows" (sandbox) lets the harness auto-run inside the sandbox and only ask at the boundary; headless/CI runs need a mode where nobody is asked and risky actions are *denied*, not silently allowed ([[dangerous-command-under-never]]).
- Approval decisions need scopes (once / session / persistent rule) and a fail-closed default, or parallel calls and caches over-grant ([[approval-scope-too-broad-in-parallel-batch]]).

## Design space
- **None** — ✔ pi: "No permission popups" ([[no-permission-prompts]]); opt-in gates via [[tool-call-gate]] extensions.
- **Edit-centric modes** (suggest / auto-edit / full-auto) — codex TS era, replaced by sandbox × approval matrix.
- **Sandbox × approval matrix** — ✔ codex: `untrusted` (ask unless a rule allows) / `on-request` (model decides when to escalate; default) / `never` (no prompts; failures returned to model; dangerous commands forbidden) / `granular` (per-category on/off; off = auto-reject).
- **Reactive mode** "ask only after a sandboxed failure" (`on-failure`) — codex had it, removed: "performs worse than `on-request`" ([[no-on-failure-approval-mode]]).
- **Who answers**: human / hooks / LLM reviewer ([[llm-approval-reviewer]]); codex precedence hooks → reviewer → user.
- **Decision scopes**: once / session cache / persistent prefix rule ([[command-rule-policy]]) / persistent network rule / persistent MCP allow; abort-turn vs deny-and-continue.
- **Policy source**: per-project default from trust ([[project-trust-gate]]), CLI flag, config layer, admin requirements allow-list (✔ codex `allowed_approval_policies`).
- **Approval key granularity**: per call id vs per turn; canonicalized command key ([[approval-key-canonicalization]]).

## Implementations
- [[codex--approval-policy-modes|codex]] — `AskForApproval::{UnlessTrusted, OnRequest (default), Granular, Never}`; `ReviewDecision` with 8 variants, default Denied; hooks → guardian → user; session cache keyed per action; CLI `--approve-for-me` / `--dangerously-bypass-approvals-and-sandbox`.

## Failures
- [[dangerous-command-under-never]]
- [[approval-scope-too-broad-in-parallel-batch]]
- [[approval-wait-surfaces-as-rejection-on-interrupt]]

## Related
[[os-level-sandbox]] · [[sandbox-escalation-retry]] · [[command-rule-policy]] · [[llm-approval-reviewer]] · [[permission-state-prompt]] · [[model-requested-permissions]] · [[tool-call-gate]] · [[project-trust-gate]] · [[layered-settings]] · [[no-permission-prompts]] · [[no-on-failure-approval-mode]] · [[isolation-strategy]]
