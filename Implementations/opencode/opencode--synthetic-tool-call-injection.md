---
type: implementation
harness: opencode
concept: synthetic-tool-call-injection
commit: ecc4916b5a
files: [packages/opencode/src/session/prompt.ts:255-449, packages/opencode/src/session/prompt.ts:451-520, packages/opencode/src/session/prompt.ts:1142-1147, packages/opencode/src/session/prompt.ts:795, packages/opencode/src/session/message-v2.ts:238-249, packages/core/src/session/runner/to-llm-message.ts:136-144]
---
[[synthetic-tool-call-injection]] in [[opencode]].

## Mechanism

### Legacy runtime
- **User `!shell`** `shellImpl` (`packages/opencode/src/session/prompt.ts:451-520`): clean up any pending revert; persist a user message with synthetic text "The following tool was executed by the user" (`:479-487`); persist an assistant message (same agent/model) with a `shell` tool part `status: "running"`, fresh `callID` (`:489-518`); run the command in-harness and complete the part. Runs inside `Effect.uninterruptibleMask`, serialized against agent runs by the session runner (`startShell`).
- **User-invoked subtask** (`/command` bound to a subagent): the user message carries a `subtask` part; the loop pops it as a task (`:1142-1147`) and `handleSubtask` (`:255-449`) creates an assistant message with a running `task` tool part `{prompt, description, subagent_type, command}`, executes `TaskTool` directly, writes completed/error state. If the subtask came from a command, a synthetic user "Summarize the task tool output above and continue with your task." follows (`:430-448`).
- Rendering: a user `subtask` part becomes text "The following tool was executed by the user" (`packages/opencode/src/session/message-v2.ts:244-249`).
- Same narration pattern for `@file` attachments: "Called the Read tool with the following input: {…}" before the content (`prompt.ts:795`, `:861`, `:936`, `:955`) → [[mention-expansion]].

### v2 runtime — tool shape dropped
- `shell` messages lower to a plain user message "Shell command: <cmd>\n\n<output>" (`packages/core/src/session/runner/to-llm-message.ts:136-144`); `session.shell` itself returns `OperationUnavailableError` at HEAD (`packages/core/src/session.ts:388`).

## Constants
| name | value | path:line |
|---|---|---|
| attribution text | "The following tool was executed by the user" | `packages/opencode/src/session/prompt.ts:484` |
| subtask follow-up | "Summarize the task tool output above and continue with your task." | `packages/opencode/src/session/prompt.ts:446` |

## Evolution
- 2025-08-27 `ad8ea82611` (#2283) synthetic user message before `!` bash execution.
- 2025-09-13 `16d66c209d` `subtask` flag for commands with a subagent.
- 2025-12-16 `5f57cee8e4` (#5650) user-invoked subtasks caused tool_use / missing thinking signature errors → synthetic summarize prompt.
- 2026-06-03 `76ee87ead8` v2 runtime: shell as user text.

## Quirks / drift
- The injected assistant message is attributed to the session's model, so later replays treat a user action as the model's own tool call; only the preceding synthetic user line disambiguates.

Contrast: pi stores user `!cmd` as a `bashExecution` custom message converted to user text and deferred to the turn boundary ([[pi--out-of-band-message-deferral|pi]]); opencode legacy fabricates the assistant tool call, v2 converges on pi's user-text shape.
