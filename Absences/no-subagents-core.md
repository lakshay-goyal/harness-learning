---
type: absence
harnesses: [pi]
---
# no-subagents-core

**What's missing**
- The stable coding agent has no built-in `task`/subagent tool. Built-in tools are `read, bash, powershell, edit, write, grep, find, ls` (`packages/coding-agent/src/core/tools/index.ts:95-105`); built-in extensions are only `llama.cpp, codemode, tool-search, mcp` (`packages/coding-agent/src/extensions/index.ts:7-14`).
- No agent-level delegation in `packages/agent` either. The loop has no concept of a child conversation.

**Evidence of decision**
- `e9935beb5` (2025-11-12, Mario Zechner), README "Sub-Agents": "**pi does not and will not support sub-agents as a built-in feature.** … Context transfer between agents is generally poor. Information gets lost, compressed, or misrepresented when passed through agent boundaries. Direct execution with full context is more effective than delegation with summarized context. If you need parallel work on independent tasks, manually run multiple `pi` sessions in different terminal tabs. You're the orchestrator."
- `3424550d2` (2025-12-17) Philosophy: "**No sub-agents.** Spawn pi instances via tmux, or build a task tool with custom tools. Full observability and steerability."
- `25cc5c7bf^:packages/coding-agent/README.md:539`: "**No sub-agents.** There's many ways to do this. Spawn pi instances via tmux, or build your own with extensions, or install a package that does it your way." The Philosophy section was deleted in `25cc5c7bf` (2026-09-22, docs refresh #9898).
- HEAD `README.md:19` / `packages/coding-agent/README.md:19` (wording from `de7e675de`, 2026-10-02): "Pi ships with powerful defaults but skips features like sub-agents and plan mode. Ask Pi to build what you want, or install a package that does it your way." This is one of only two skipped features still named at HEAD.
- Governance: "**pi's core is minimal** … PRs that bloat the core will likely be rejected." (`CONTRIBUTING.md:7-9`).

**Opt-in replacement**
- `packages/coding-agent/examples/extensions/subagent/` (added `eb1d08a5f`, 2025-12-18, PR #215, Nico Bailon) → [[subagent-as-subprocess]].
  - Each invocation spawns a separate `pi` process with an isolated context window (`subagent/index.ts:1-13`). Args are `--mode json -p --no-session` (`:300`); the role prompt is passed via `--append-system-prompt` (`:300-338`).
  - Modes: single, parallel, and chain with a `{previous}` placeholder (`:7-10`). Limits `MAX_PARALLEL_TASKS = 8`, `MAX_CONCURRENCY = 4`, `PER_TASK_OUTPUT_CAP = 50 KiB` (`:33-36`).
  - Agents are markdown files with `tools` and `model` frontmatter. It ships scout, planner, reviewer and worker; `agents/scout.md` uses `model: claude-haiku-4-5` with tools `read, grep, find, ls, bash`. Scout's output contract is written for a reader "who has NOT seen the files you explored" (`agents/scout.md:10`).
  - Project-local `.pi/agents` are opt-in via `agentScope` and gated by a prompt: "Project agents are repo-controlled. Only continue for trusted repositories." (`subagent/index.ts:484-545`, `subagent/README.md:55-65`). The prompt is skipped in trusted projects (`8af7690c4`).
- `handoff.ts` example moves distilled context into a fresh session instead of delegating → [[session-handoff]].
- Tool-level orchestration without agent-level delegation: `ctx.executeTool()` with `parentToolCallId` and a bounded `nestedCalls` record (`packages/coding-agent/docs/extensions.md:148`, `8562bcf66`) → [[nested-tool-calls]], [[code-mode]].

**History**
- 2025-11-12 "will not" (`e9935beb5`) → 2025-12-17 Philosophy one-liner (`3424550d2`) → 2025-12-18 example extension (`eb1d08a5f`) → 2026-09-22 Philosophy deleted (`25cc5c7bf`) → 2026-10-02 still skipped in README (`de7e675de`).
- **Partial reversal in the durable runtime** ([[task-owned-subagent]]):
  - `packages/durable` gained subagent primitives in `2532a0bef` (2026-09-29, "conversation abort, ownership cascades, and subagent handles (Package 18)"): task-owned conversations with `ownership: { kind: "task", taskId }` and abort cascades (`packages/durable/README.md:414-426`). `03180653c` added structured concurrency.
  - The kernel itself ships no subagent tool; it is a pattern (`test/examples/22-subagent-foreground.ts`, `23-subagent-background.ts`).
  - The experimental durable TUI does ship a foreground `subagent` tool (`packages/coding-agent/src/experimental/durable/subagent.ts:24-55`). It is `replay: "safe"` (`:33`); a rerun finds the existing child via `scanConversations({ownerTaskId})` (`:36-37`). The child removes the Subagent extension so it cannot recurse (`:40`), and uses `requestId: subagent:${taskId}` (`:45`).
  - Background variants: the kernel example `packages/durable/test/examples/23-subagent-background.ts`, which `37c9d0d20` (2026-09-30) made not-rerun-after-crash because "a repeated stop could stop newer work"; and the vacation demo's `research` tool (`packages/coding-agent/src/experimental/vacation/vacation.ts:105-127`, `395315f48`, 2026-10-01).
  - The durable package is marked "**Experimental.** The API changes without notice" (`packages/durable/README.md:3`). The experimental services do not expose subagents (`experimental/services/README.md:38-45`).
- Spec footgun: a subagent tool must not report child usage, or the subtree sum counts it twice (`packages/durable/docs/spec.md:4634-4636`).

**Implication**
- pi judges the cost of delegation (context lost at the boundary) to be higher than its benefit. Orchestration goes to the human (tabs/tmux) or to an extension that reuses the CLI as a subprocess, so process isolation is the context isolation.
- "No subagents" is a product stance for the stable coding agent, not a runtime limit. Once crash-safe ownership existed ([[durable-execution]], [[crash-safe-tool-replay]]), subagents became cheap to express. Watch for promotion to stable (unverified intent; `5609b0d6c` experimental TUI on pi-durable).

**codex** — *not absent*: multi-agent tools since `86f81ca010` / `623707ab58` 2026-01-12 ("collab" spawn/wait/close), renamed multi_agent 2026-02-16 `e41536944e`, V2 communication pattern `773fbf56a4` 2026-03-24; `multi_agent` Stable default **on** (`codex-rs/features/src/lib.rs:1422-1427`) → [[in-process-subagent-threads]]; earlier review delegated to a sub-Codex instance `13e1d0362d` 2025-10-29 → [[review-subagent]]; CSV fan-out jobs `dcab40123f` 2026-02-24 later removed → [[no-csv-fanout-jobs]]; cloud best-of-N delegation → [[cloud-task-delegation]].
**opencode contrast**: implements it: `task` tool, child sessions, built-in `general` / `explore` subagents, experimental background mode, depth cap 1 (`packages/opencode/src/tool/task.ts:83-345`; `packages/opencode/src/agent/agent.ts:182-218`) — see [[task-owned-subagent]] / [[agent-profiles]] / [[builtin-subagents-vs-none]].

Related: [[subagent-as-subprocess]] · [[task-owned-subagent]] · [[session-handoff]] · [[nested-tool-calls]] · [[replaceable-builtin-extension]] · [[no-plan-mode]] · [[Absences]] · [[in-process-subagent-threads]] · [[subagent-result-mailbox]] · [[review-subagent]] · [[cloud-task-delegation]] · [[builtin-subagents-vs-none]]
