---
type: group
group: 09-subagents
---
Failures whose first concept is in [[Subagents]].

- [[subagent-output-truncated]] — Parallel subagents returned 100-char previews to the parent model, losing results and diagnostics.
- [[subagent-config-not-inherited]] — Subagents ignored parent model/thinking/tools; array-form `tools` rejected; wrong agents dir.
- [[subagent-prompt-leaks-host-paths]] — Child invocation leaked Bun virtual-FS paths / used a different `pi` build.
- [[ownership-cancellation-races]] — Durable ownership-tree abort cascades raced with completion, orphaning and reopen (pi); parent abort left subtask children running and cancels were lost during shell/run start (opencode).

## Plan mode (opencode)
- [[read-only-mode-bypass-via-subagent]] — plan mode delegated edits to the edit-capable `general` subagent.
- [[read-only-mode-bypassed-via-shell]] — plan mode edited files with `sed`/`tee`/`echo` because bash stayed allowed.
- [[stale-mode-reminder-persists]] — plan-mode reminders kept the model read-only after switching to build.
- [[model-initiated-mode-switch]] — the model called `plan_enter` mid-task and dropped its own write ability.

## Task tool (opencode)
- [[unbounded-subagent-nesting]] — subagents spawned subagents without a depth cap.
- [[model-polls-background-work]] — a `task_status` tool made the model sleep and poll background subagents.
- [[subagent-error-reported-as-success]] — failed child runs returned their last text as success.
- [[delegated-work-duplicated]] — the parent redid work it had delegated.
- [[ambiguous-tool-parameter-name]] — GPT models misused `session_id`; renamed `task_id` with "only set if you mean to resume".

Related cross-group: [[non-idempotent-tool-replayed-after-crash]] (background subagent stop replayed after crash, 03-tools) · [[parallel-side-requests-single-slot-provider]] (05-context) · [[proxied-stream-option-loss]] (01-loop).
