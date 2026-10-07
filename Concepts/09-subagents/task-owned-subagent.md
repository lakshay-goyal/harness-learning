---
type: concept
stage: subagents
tier: candidate
aliases: [durable subagent tool, "ownership: {kind: task}", completing hold, structured-concurrency-ownership, background task, failFast, allSettled, keyed-service-generations, research task, scanConversations ownerTaskId, task tool (opencode), subagent_type, task_id, subagent_depth, OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS, BackgroundJob.Service]
harnesses: [pi, opencode]
---
A subagent is a child conversation owned by the tool task that spawned it, under structured concurrency: abort flows down, the owner cannot finish before owned work drains, background tasks are explicit boundaries.

## Why
- Subprocess subagents are invisible to the harness's lifecycle: aborts, idle, crash recovery and accounting must be reimplemented per tool.
- Without an ownership tree, cancellations race (children outlive or miss parent aborts) ([[ownership-cancellation-races]]).
- Crash-resumable runtimes need idempotent spawn/submit so a replayed tool call finds the same child, and must not replay non-idempotent control operations ([[non-idempotent-tool-replayed-after-crash]]).
- Usage accounting double counts if the tool also reports the child's spend.

## Design space
- **Ownership**: task-owned child conversation (pi-durable) · ownerless + manual bookkeeping · subprocess ([[subagent-as-subprocess]]).
- **Foreground vs background**: tool waits for child answer (holds the run) · background anchor task that reports back as a follow-up message (pi-durable vacation / example 23).
- **Idempotence**: ownership index lookup + per-task `requestId` on submissions → `replay:"safe"` · not replay-safe for stop/kill.
- **Join policy**: `failFast` (abort siblings on first failure) · `allSettled`.
- **Abort order**: bottom-up so compensating handlers see final child outcomes.
- **Recursion**: child config removes the subagent extension.
- **Interrogation**: child conversation survives the call so users can switch to it.
- **Exposure to remote UIs**: keyed service instances per conversation with generation fencing (planned).
- **Child placement**: child session in the same process linked by `parentID`, resumable by `task_id` (opencode).
- **Depth**: default cap 1 via `subagent_depth` (opencode, since `285d315b4e`) — see [[unbounded-subagent-nesting]].
- **Background completion**: poll tool (opencode `task_status`, removed) vs synthetic user message that starts a new parent turn (opencode now); follow-up calls with the same id extend a running job.
- **Promotion**: user moves a blocking foreground subagent to the background (opencode experimental route).
- **Durability of the job registry**: deliberately in-process, lost on restart (opencode `BackgroundJob`) vs durable ownership tree (pi-durable).
- **Permission inheritance**: parent session denies + `external_directory` only; parent agent restrictions not inherited (opencode HEAD) — see [[read-only-mode-bypass-via-subagent]].
- **Result contract**: last child text wrapped in `<task_result>`; child error or failed last tool fails the call with a resumable id (opencode).

## Implementations
- [[pi--task-owned-subagent|pi]] — pi-durable ownership tree (Package 18/19) + experimental TUI `subagent` tool and vacation `research` background subagent.
- [[opencode--task-owned-subagent|opencode]] — legacy `task` tool: child session per call, `subagent_depth` 1, `task_id` resume, `<task_result>` XML, experimental background jobs with push completion; no task tool in v2 yet.

## Failures
- [[background-job-without-observation-tool]]
- [[ownership-cancellation-races]]
- [[non-idempotent-tool-replayed-after-crash]]
- [[unbounded-subagent-nesting]]
- [[model-polls-background-work]]
- [[subagent-error-reported-as-success]]
- [[delegated-work-duplicated]]
- [[ambiguous-tool-parameter-name]]
- [[read-only-mode-bypass-via-subagent]]
- [[subagent-config-not-inherited]]
- [[approval-wait-without-responder]]

## Tradeoffs
- [[builtin-subagents-vs-none]]

## Related
[[subagent-as-subprocess]] · [[durable-execution]] · [[crash-safe-tool-replay]] · [[abort-propagation]] · [[follow-up-queue]] · [[run-settlement]] · [[client-server-session-split]] · [[replicated-state]] · [[no-subagents-core]] · [[agent-profiles]] · [[plan-mode]] · [[permission-ruleset]]
