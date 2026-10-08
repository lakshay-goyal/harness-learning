---
type: absence
harnesses: [opencode]
---
# no-background-task-polling

Background subagents notify; there is no status tool to poll.

**What's missing**
- No `task_status` tool. A `task` call with `background: true` returns at once, and the result says "You will be notified when it completes. DO NOT sleep, poll, or proactively check on its progress" (`packages/opencode/src/tool/task.ts:58-61`). Completion is injected into the parent session as a synthetic user message with the task XML.

**Evidence of decision**
- `22de34c4de` (2026-05-14, #27084) added experimental background subagents **with** `task_status`: params `task_id`, `wait`, `timeout_ms` (`dabf2dc013^:packages/opencode/src/tool/task_status.txt:1-8`).
- `dabf2dc013` (2026-05-25, #29179) "remove the need for polling from experimental background agents" deleted it (179 lines), 11 days after landing.
- Same principle applied to bash in v2: background bash removed because "The model has no registered observation or cancellation tool" (`specs/v2/schema-changelog.md:697`) → [[no-background-bash]].

**Implication**
- A status tool invites busy-waiting: the model spends turns asking "done yet?". Push completion as a message and forbid polling in the tool result itself. The registry behind it is process-local and "intentionally not durable" (`packages/core/src/background-job.ts:113-118`).
- Failure → [[model-polls-background-work]].

Related: [[task-owned-subagent]] · [[out-of-band-message-deferral]] · [[tool-description-design]] · [[no-background-bash]] · [[builtin-subagents-vs-none]] · [[opencode]] · [[Absences]]
