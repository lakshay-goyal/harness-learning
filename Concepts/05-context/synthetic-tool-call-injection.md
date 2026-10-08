---
type: concept
stage: context
tier: variant
aliases: [SubtaskPart, handleSubtask, shellImpl, "The following tool was executed by the user", "Summarize the task tool output above and continue with your task."]
harnesses: [opencode]
---
Work the user triggers directly, such as a `!` shell command or a subagent command, is recorded as an assistant tool call plus its result, so the model sees it in its own tool shape.

## Why
- The model already knows how to read tool results; user-run output pasted as free text is weaker evidence and easy to mistake for an instruction.
- The transcript must stay provider-valid: an assistant tool call needs a preceding user turn and a matching result, and some providers require signed reasoning on assistant turns (opencode `5f57cee8e4` 2025-12-16 "user invoked subtasks causing tool_use or missing thinking signature").
- Downside: the model sees tool calls it never made; its own-history reasoning ("I ran X") becomes false, so a synthetic user line has to say who ran it.

## Design space
- **Shape**: synthetic user note + assistant tool call + result (opencode legacy, for `!shell` and `/command` subtasks) · plain user message with the output (opencode v2 "Shell command: …"; pi `bashExecution` custom role → user text, see [[message-conversion-layer]]) · tool-result-only.
- **Attribution text**: "The following tool was executed by the user" (opencode legacy, before the call) · none.
- **Follow-up nudge**: synthetic user "Summarize the task tool output above and continue with your task." after a command-run subtask (opencode legacy).
- **Attachments**: `@file` mentions rendered as fake tool narration "Called the Read tool with the following input: …" followed by the content (opencode legacy) → [[mention-expansion]].
- **Timing**: while the agent is busy the injection must wait for the turn boundary ([[out-of-band-message-deferral]]); opencode legacy serializes shell vs run in a per-session runner state machine.

## Implementations
- [[opencode--synthetic-tool-call-injection|opencode]] — legacy `shellImpl` and `handleSubtask` write a synthetic user text + assistant message with a running `shell`/`task` tool part, execute it in-harness, then complete it; v2 drops the tool shape (shell becomes a user message).

## Failures
- Cross-group: [[orphaned-tool-calls-and-results]] (a user-run tool interrupted mid-way must still be answered; legacy renders pending/running parts as "[Tool execution was interrupted]")

## Related
[[out-of-band-message-deferral]] · [[message-conversion-layer]] · [[mention-expansion]] · [[shell-execution]] · [[task-owned-subagent]] · [[transcript-replay-repair]]
