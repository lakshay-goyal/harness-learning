---
type: implementation
harness: opencode
concept: task-owned-subagent
commit: ecc4916b5a
files: [packages/opencode/src/tool/task.ts:24-61, packages/opencode/src/tool/task.ts:96-175, packages/opencode/src/tool/task.ts:200-358, packages/opencode/src/agent/subagent-permissions.ts:14-27, packages/opencode/src/tool/registry.ts:265-278, packages/core/src/background-job.ts:7, packages/core/src/background-job.ts:112-118, packages/opencode/src/server/routes/instance/httpapi/handlers/experimental.ts:159-172]
---
[[task-owned-subagent]] in [[opencode]].

## Mechanism
### Legacy runtime (`task` tool)
- **Child = session in the same process**: `sessions.create({parentID, title: "<description> (@<agent> subagent)", agent, permission})` (`packages/opencode/src/tool/task.ts:156-172`). Model = subagent's own model, else the parent message's model and variant (`packages/opencode/src/tool/task.ts:181-184`).
- **Resume**: `task_id` reuses an existing child session; parameter description: "This should only be set if you mean to resume a previous task" (`packages/opencode/src/tool/task.ts:47-50`).
- **Depth cap**: walk `parentID` to the root; fail with `Subagent depth limit reached (N). Increase "subagent_depth"…` when depth ≥ `subagent_depth ?? 1`, so subagents cannot spawn subagents by default (`packages/opencode/src/tool/task.ts:104-117`; `285d315b4e`) ([[unbounded-subagent-nesting]]).
- **Permission**: `task` permission asked with pattern = subagent name and "always" `["*"]` unless `bypassAgentCheck` (user-invoked) (`packages/opencode/src/tool/task.ts:119-129`). Child session ruleset (`packages/opencode/src/agent/subagent-permissions.ts:14-27`): parent **session** deny + `external_directory` rules; `todowrite` and `task` denied unless the subagent's own ruleset mentions them; `experimental.primary_tools` denied (`packages/opencode/src/tool/task.ts:143-155`). The parent **agent's** restrictions are deliberately not inherited ("Parent agent restrictions only govern that agent").
- **Advertising**: the tool description gets "Available agent types and the tools they have access to:" + non-primary agents the caller's ruleset does not deny, descriptions only (`packages/opencode/src/tool/registry.ts:265-278`).
- **Result**: last text part wrapped as `<task id=… state=…><task_result>…</task_result></task>` (`<task_error>` on error) via `renderOutput` (`packages/opencode/src/tool/task.ts:64-75`). Child assistant error or a last errored tool part → tool fails with `Subagent failed (task_id: …): <error>` (`packages/opencode/src/tool/task.ts:213-224`) ([[subagent-error-reported-as-success]]).
- **Abort**: parent abort signal cancels the child session; interrupt also cancels the background job (`packages/opencode/src/tool/task.ts:324-357`).
- **Background** (flag `OPENCODE_EXPERIMENTAL_BACKGROUND_SUBAGENTS`, `packages/opencode/src/tool/task.ts:98-102`):
  - Returns at once with "The task is working in the background. You will be notified automatically when it finishes. DO NOT sleep, poll for progress, ask the task for status, or duplicate this task's work…" (`packages/opencode/src/tool/task.ts:31-35`).
  - Completion injects a synthetic user prompt into the parent session (`Background task completed: <description>` + task XML), which starts a new parent turn (`packages/opencode/src/tool/task.ts:227-264`) ([[model-polls-background-work]]).
  - Same `task_id` while running → `background.extend` appends work, serialized after the previous run (`packages/opencode/src/tool/task.ts:267-282`).
  - A foreground task races `wait` against `waitForPromotion`; the experimental route `POST …/session/:id/background` promotes all running foreground tasks of a session (`packages/opencode/src/tool/task.ts:334-338`; `packages/opencode/src/server/routes/instance/httpapi/handlers/experimental.ts:159-172`).
- **Job registry** (`packages/core/src/background-job.ts`): status `running|completed|error|cancelled` (`packages/core/src/background-job.ts:7`); "intentionally not durable: process restart or owner-scope closure loses status and interrupts live work" (`packages/core/src/background-job.ts:112-118`). Only `task` uses it; deleting a session cancels its jobs.

### v2 runtime
- No `task` tool in `packages/core/src/tool/` at HEAD; delegation is legacy-only. `general`/`explore` agents are already defined in v2 (`packages/core/src/plugin/agent.ts:152-160`).

## Constants
| name | value | path:line |
|---|---|---|
| `subagent_depth` default | 1 | `packages/opencode/src/tool/task.ts:111` |
| concurrency cap | none found | — |
| task output cap | generic tool truncation (per-agent limits) | `packages/opencode/src/tool/tool.ts:131-142` |

## Evolution
- 2025-07-20 `a524fc545c` `session_complete` hook no longer fires for subagent sessions. 2025-08-19 `a2db58f125` `--continue` cannot resume a subagent session.
- 2025-08-12 `790e9947bd` task.txt examples marked fictional (model called them); 2026-05-16 `548648a3d9` examples removed.
- 2025-12-16 `5f57cee8e4` user-invoked subtasks left `tool_use` without result.
- 2026-01-13 `5d37e58d34` nested subagents ignored their own `task` permission.
- 2026-01-15 `8b08d340ac`, 2026-01-16 `08b94a6890` subtasks changed the parent's selected model/agent ([[subagent-config-not-inherited]]).
- 2026-02-05 `64e2bf8bf0` `session_id` → `task_id` with "only set if you mean to resume" for GPT tool-call failures.
- 2026-04-30 `d7701dbfb6` child keeps parent session denies + `external_directory`.
- 2026-05-09 `b8ca71d309` → 2026-05-12 `c4e676b8a0` → 2026-06-10 `3ad6923c61`: parent-agent deny inheritance added, narrowed, removed ([[read-only-mode-bypass-via-subagent]]).
- 2026-05-12 `8feb4a31c7` background job service; 2026-05-14 `22de34c4de` background subagents; 2026-05-25 `dabf2dc013` `task_status` polling tool removed; 2026-06-04 `cc9b73b0bd` anti-polling prompt (escalated to "DO NOT sleep, poll…" `b9131aa69c` 2026-06-06), `70bb710715` "do not duplicate that work yourself" ([[delegated-work-duplicated]]).
- 2026-07-15 `285d315b4e` depth cap.
- 2026-08-20 `c313504c82`, 2026-08-21 `35fe5b7212` subagent errors surfaced with resumable `task_id`.

## Quirks / drift
- Delegation policy is opposite across per-model prompts: anthropic.txt pushes the Task tool ("You should proactively use the Task tool…", "VERY IMPORTANT: When exploring the codebase … CRITICAL that you use the Task tool"), gpt-astra says "Do not spawn subagents unless the user … explicitly ask" (`packages/opencode/src/session/prompt/anthropic.txt:79-94`; `packages/opencode/src/session/prompt/gpt-astra.txt:44-46`).
- `general` is described as "execute multiple units of work in parallel" but there is no concurrency cap (`packages/opencode/src/agent/agent.ts:184`).

pi contrast: pi-durable child conversations owned by a tool task with structured-concurrency abort ([[pi--task-owned-subagent|pi]]); pi core has no subagents ([[no-subagents-core]]).
