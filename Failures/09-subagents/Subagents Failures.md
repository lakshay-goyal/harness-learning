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

## Delegation prompting
- [[over-eager-delegation]] — Spawned sub-agents for "be thorough" requests; fix: explicit-authorization rule with near-miss phrasings (codex).
- [[subagent-model-downgrade]] — "A mini model can solve many tasks…" made the parent pick older models for children (codex).

## Mailbox & capacity
- [[wait-misses-already-queued-result]] — `wait_agent` missed mail queued before subscribing and slept the full timeout (codex).
- [[blocking-wait-ignores-user-steer]] — A long `wait_agent` held the turn while a user steer waited (codex).
- [[subagent-eviction-loses-mail]] — LRU unloading of idle agents raced/dropped queued messages (codex).
- [[depth-capped-agent-still-has-tools]] — Max-depth agent still saw spawn tools that always failed (codex).

## Roles & review
- [[subagent-role-escalates-authority]] — Role config layers could widen permissions/providers/MCP; fix: narrowing-only allowlist (codex).
- [[review-agent-edits-code]] — `/review` child changed code; read-only sandbox broke review; fix: config-enforced lockdown (codex).
- [[side-task-result-invisible-to-parent]] — Review findings shown in UI but absent from the main agent's history (codex).

Related cross-group: [[forked-child-inherits-parent-tool-noise]] (08-state) · [[non-idempotent-tool-replayed-after-crash]] (background subagent stop replayed after crash, 03-tools) · [[parallel-side-requests-single-slot-provider]] (05-context) · [[proxied-stream-option-loss]] (01-loop).
