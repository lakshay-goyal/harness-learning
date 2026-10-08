---
type: concept
stage: subagents
tier: variant
aliases: [codex cloud, Codex Cloud tasks, CloudBackend, cloud-tasks-client, cloud-tasks-mock-client, cloud-client, ThreadService, "--attempts", best-of-N attempts]
harnesses: [codex]
---
Hand a task to a remote hosted agent environment (optionally best-of-N attempts), then browse its diffs/messages and apply one attempt locally.

## Why
- Long or parallel work shouldn't occupy the local machine or session; hosted environments run in the background and in parallel.
- Several independent attempts at the same task trade cost for quality; the human picks the best diff.
- Remote control RPCs that time out may have taken effect — retries must not duplicate admission.

## Design space
- **Local child** ([[in-process-subagent-threads]], [[subagent-as-subprocess]]) · remote hosted task with diff apply (✔ codex).
- **Attempts**: one · best-of-N 1..4 with sibling-attempt browsing (✔ codex).
- **Result intake**: final text · preflight + apply diff locally (✔ codex `apply_task_preflight`, `apply_task`).
- **RPC semantics**: retried · never retried, attach is observation-only — detaching neither interrupts nor answers approvals (✔ codex gRPC client).

## Implementations
- [[codex--cloud-task-delegation|codex]] — `codex cloud exec|status|list|apply|diff` (`codex-rs/cloud-tasks`), backend trait + mock (`codex-rs/cloud-tasks-client`), native gRPC ThreadService client (`codex-rs/cloud-client`).

## Failures
none recorded.

## Related
[[in-process-subagent-threads]] · [[client-server-session-split]] · [[git-worktree-isolation]] · [[builtin-subagents-vs-none]]
