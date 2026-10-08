---
type: concept
stage: messages
tier: variant
aliases: [PermissionsInstructions, permissions developer message, permissions_instructions.rs, approval_policy templates, sandbox_mode templates, on_request.md, "# Escalation Requests", Approved command prefix saved, dynamic sandbox prompt]
harnesses: [codex]
---
The active sandbox / approval / network policy is rendered into a dynamic model-facing message (re-emitted or incrementally amended when the policy changes) that tells the model what it may do, which failures are sandbox-induced, and exactly which channel to use to ask for more.

## Why
- A model that doesn't know it is sandboxed reads `Could not resolve host` as a flaky network and gives up ([[sandbox-network-error-not-escalated]], [[sandbox-failure-misread-as-transient]]).
- Without a named escalation channel the model asks in prose, stalling headless runs ([[approval-asked-in-prose]]), or routes around denials with other tools ([[approval-circumvention]]).
- Persistent rules drafted by the model need prompt-level ban lists ([[model-proposed-rule-too-broad]]); shared worktrees need "don't touch changes you didn't make" ([[destructive-git-on-user-changes]]).
- Static prompt text goes stale when the policy changes mid-session; a fallback assumption ("assume workspace-write, network ON") was wrong for many configs.

## Design space
- **Nothing in the prompt** — pi: no sandbox/approvals to describe ([[minimal-system-prompt]]).
- **Static section in the base prompt** — codex 2025-08 → 2026-01 (~35-line "Sandbox and approvals" in every per-model prompt).
- **Dynamic developer message per policy**, re-sent on change (✔ codex since `87f7226cca`), with **incremental notes** for newly saved rules instead of re-sending ([[world-state-diff-injection]]).
- **Templating**: one template per sandbox mode and per approval policy + inline sections (request_permissions tool, auto-review suffix, approved prefixes, denied reads, granular categories) (✔ codex); per-model catalog overrides ([[per-model-system-prompt]]).
- **Size bound**: cap listed paths with an explicit "restrictions still apply" omission notice (✔ codex 32 KiB).
- **Vocabulary tuning**: network "restricted/enabled" beat "ON/OFF" (codex `90d892f4fd`).

## Implementations
- [[codex--permission-state-prompt|codex]] — `<permissions instructions>` developer fragment from `codex-rs/prompts/templates/permissions/{sandbox_mode,approval_policy}/*.md` + inline constants; incremental "Approved command prefix saved:" notes.

## Failures
- [[approval-asked-in-prose]]
- [[approval-circumvention]]
- [[sandbox-network-error-not-escalated]]
- [[model-proposed-rule-too-broad]]
- [[destructive-git-on-user-changes]]
- [[permission-context-reinjected-repeatedly]] (06-caching)

## Related
[[approval-policy-modes]] · [[os-level-sandbox]] · [[sandbox-escalation-retry]] · [[model-requested-permissions]] · [[command-rule-policy]] · [[message-role-layering]] · [[xml-prompt-boundaries]] · [[world-state-diff-injection]] · [[per-model-system-prompt]] · [[dynamic-tool-guidelines]] · [[plan-mode]]
