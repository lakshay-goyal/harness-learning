---
type: tradeoff
concepts: [task-owned-subagent, subagent-as-subprocess, agent-profiles, session-handoff]
harnesses: [pi, opencode]
---
# builtin-subagents-vs-none

**Axis**: does the model get a built-in tool to delegate work to a child agent with its own context?

| option | pi | opencode | evidence |
|---|---|---|---|
| None; the human orchestrates (tabs, tmux) | ✅ stable | ❌ | pi: "Context transfer between agents is generally poor … You're the orchestrator." (`e9935beb5`); HEAD still skips it (`README.md:19`) → [[no-subagents-core]] |
| Subagent as a separate CLI process (extension) | ✅ `examples/extensions/subagent/` (8 parallel, 4 concurrent, 50 KiB cap) | — | `subagent/index.ts:33-36,300` → [[subagent-as-subprocess]] |
| In-process child session via `task` tool | experimental durable only (`packages/coding-agent/src/experimental/durable/subagent.ts:24-55`) | ✅ default: `general`, `explore`; resumable `task_id`; result wrapped in `<task_result>` | opencode: `packages/opencode/src/tool/task.ts:83-345`; `packages/opencode/src/agent/agent.ts:182-218` → [[task-owned-subagent]] |
| Nesting limit | durable child removes its own subagent extension (`subagent.ts:40`) | `subagent_depth ?? 1` since `285d315b4e` 2026-07-15 (unbounded before) | `packages/opencode/src/tool/task.ts:106-115` → [[unbounded-subagent-nesting]] |
| Background subagents | durable examples only | experimental flag; push completion, no polling tool | `packages/opencode/src/tool/task.ts:58-61`; `dabf2dc013` → [[no-background-task-polling]] |
| Permission inheritance | n/a | child gets parent session denies + `todowrite`/`task` deny, not the parent agent's restrictions | `packages/opencode/src/agent/subagent-permissions.ts:4-26` |

**When each wins**
- **No subagents (pi)**: tasks where full context beats summarized context, and where the human wants observability of every step. No double-counted usage, no permission-inheritance bugs, no recursion.
- **Built-in subagents (opencode)**: wide read-only exploration (`explore` keeps the parent's window clean), parallel independent work, and cheaper models for scouting. Costs: context loss at the boundary, mode-escape bugs (plan → general), recursion that needs a depth cap, and async coordination (polling vs push).

Related: [[no-subagents-core]] · [[plan-mode-vs-none]] · [[agent-profiles]] · [[pi]] · [[opencode]] · [[Tradeoffs]]
