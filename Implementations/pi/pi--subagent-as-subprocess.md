---
type: implementation
harness: pi
concept: subagent-as-subprocess
commit: b30a6dd77
files: [packages/coding-agent/examples/extensions/subagent/index.ts:33, packages/coding-agent/examples/extensions/subagent/index.ts:249, packages/coding-agent/examples/extensions/subagent/index.ts:300, packages/coding-agent/examples/extensions/subagent/index.ts:472, packages/coding-agent/examples/extensions/subagent/agents.ts:42, packages/coding-agent/examples/extensions/subagent/agents/scout.md:10, packages/coding-agent/examples/extensions/subagent/README.md:55]
---
[[subagent-as-subprocess]] in [[pi]] — **example extension only**, not core ([[no-subagents-core]]).

## Mechanism
- Stance: core ships no task/subagent tool (built-ins `read, bash, powershell, edit, write, grep, find, ls`, `packages/coding-agent/src/core/tools/index.ts:95-105`; built-in extensions `llama.cpp, codemode, tool-search, mcp`, `src/extensions/index.ts:7-14`). README: "skips features like sub-agents and plan mode. Ask Pi to build what you want, or install a package" (`packages/coding-agent/README.md:19`). Original rationale (`e9935beb5`, 2025-11-12): "Context transfer between agents is generally poor… Direct execution with full context is more effective than delegation with summarized context… You're the orchestrator." Dec-2025: "Spawn pi instances via tmux, or build a task tool with custom tools" (`3424550d2`).
- Opt-in replacement: `examples/extensions/subagent/` (`eb1d08a5f`, 2025-12-18, PR #215). Registers model-callable tool `subagent` (`index.ts:472-481`) → [[plugin-tools]].
- **Spawn** (`runSingleAgent`, `index.ts:272-440`): args `["--mode","json","-p","--no-session"]` (`:300`) → same CLI headless, JSON event stream on stdout, print mode, no session file ([[headless-rpc-mode]], [[agent-event-stream]]). Model = agent's `model` else dispatcher's `provider/id`; thinking level inherited **only** when the agent has no model (`:301-306`); `--tools a,b` from frontmatter (`:307`); role prompt written to `mkdtemp(pi-subagent-*)/prompt-<name>.md` mode `0600` under `withFileMutationQueue` and passed via `--append-system-prompt <file>` (`:239-247`, `:334-339`) → [[system-prompt-override]]; task as final arg `Task: <task>` (`:341`); `spawn(..., {shell:false, stdio:["ignore","pipe","pipe"], cwd})` (`:345-350`); temp file/dir removed in `finally` (`:426-439`).
- **Self-reinvocation** `getPiInvocation` (`index.ts:249-263`): reuse `process.execPath` + current script if it exists and is **not** a Bun virtual path `/$bunfs/root/` (compiled binary); compiled non-node/bun exec → run `execPath` directly; else `pi` on PATH (`7c92bb815` #2464; `462b3d21e` #3002).
- **Result capture**: parse stdout JSONL; on `message_end` push message, sum usage (input/output/cacheRead/cacheWrite/cost), `turns++`, `contextTokens = totalTokens`, keep `stopReason`/`errorMessage` (`index.ts:353-388`). `getFinalOutput` = **last assistant text part** (`:170-180`). Failure = exit≠0 or `stopReason` error/aborted (`:182-184`); failed output = errorMessage || stderr || final text || "(no output)" (`:186-191`).
- **Modes** (exactly one, else error listing agents, `index.ts:493-518`):
  - single `{agent, task, cwd?}` → `Agent <stopReason>: …` + `isError` on failure (`:688-714`);
  - parallel `{tasks:[…]}` ≤ `MAX_PARALLEL_TASKS` 8, run via `mapWithConcurrencyLimit` worker pool of `MAX_CONCURRENCY` 4 (`:219-237`, `:604-686`); parent gets `Parallel: k/n succeeded` + per-task `### [agent] completed|failed (reason)` blocks, each capped at `PER_TASK_OUTPUT_CAP` 50 KiB with `[Output truncated: N bytes omitted. Full output preserved in tool details.]` (`:193-202`, `:669-681`); full results in `details` ([[structured-tool-output]]);
  - chain `{chain:[…]}` sequential; `{previous}` replaced by prior step's final text; stops at first failing step "Chain stopped at step i (agent): …" (`:550-602`).
- **Live progress**: `onUpdate` partial results with details; parallel shows "Parallel: d/n done, r running..." (`index.ts:632-643`); TUI renderer collapsed (last 10 items, `COLLAPSED_ITEM_COUNT`) / expanded markdown (`:35`; `packages/coding-agent/examples/extensions/subagent/README.md:99-123`).
- **Abort**: tool signal → `SIGTERM`, `SIGKILL` after 5 s, then throw "Subagent was aborted" (`index.ts:410-424`) → [[abort-propagation]].
- **Agent definitions** (`agents.ts`): markdown + YAML frontmatter `name`, `description` (required), `tools` (string "a, b" or YAML array — both accepted, bad value → no tools rather than throw, `agents.ts:42-59`), `model`; user dir `~/.pi/agent/agents` (via `getAgentDir()`), project dir = nearest ancestor `.pi/agents` (`agents.ts:116-125`); discovered fresh each invocation (`packages/coding-agent/examples/extensions/subagent/README.md:176`).
- **Trust**: `agentScope` default `"user"`; `"project"|"both"` opt-in; project agents override same-name user agents; interactive confirm "Project agents are repo-controlled. Only continue for trusted repositories." skipped when `ctx.isProjectTrusted()` (`index.ts:520-548`; `packages/coding-agent/examples/extensions/subagent/README.md:55-65`; `8af7690c4`) → [[project-trust-gate]].
- **Role prompts** (sample agents): `scout` (haiku-4-5; read, grep, find, ls, bash) "return structured findings that another agent can use without re-reading everything. Your output will be passed to an agent who has NOT seen the files you explored." (`agents/scout.md:8-10`), output sections Files Retrieved (exact line ranges) / Key Code / Architecture / Start Here; `planner` (sonnet-4-5; read-only tools) "You must NOT make any changes… The worker agent will execute it verbatim" (`planner.md:10,37`); `reviewer` (sonnet-4-5) "Bash is for read-only commands only… Assume tool permissions are not perfectly enforceable" (`reviewer.md:10-11`); `worker` (sonnet-4-5, all default tools) "isolated context window… without polluting the main conversation", handoff list of changed files/functions (`worker.md:7-24`).
- **Workflow prompt templates** (`prompts/*.md`, `$@` args): `/implement` scout→planner→worker; `/scout-and-plan` scout→planner ("Do NOT implement"); `/implement-and-review` worker→reviewer→worker — all instruct the model to call `subagent` with `chain` + `{previous}` (`prompts/implement.md:4-10`) → [[prompt-template-expansion]].

## Constants
| name | value | path:line |
|---|---|---|
| `MAX_PARALLEL_TASKS` | 8 | `examples/extensions/subagent/index.ts:33` |
| `MAX_CONCURRENCY` | 4 | `index.ts:34` |
| `COLLAPSED_ITEM_COUNT` | 10 | `index.ts:35` |
| `PER_TASK_OUTPUT_CAP` | 50 KiB (= tool truncation limit) | `index.ts:36` |
| kill escalation | SIGTERM → SIGKILL 5000 ms | `index.ts:411-416` |
| child args | `--mode json -p --no-session` | `index.ts:300` |

## Evolution
- 2025-11-12 `e9935beb5` README "pi does not and will not support sub-agents as a built-in feature".
- 2025-12-18 `eb1d08a5f` subagent orchestration example (#215); 2025-12-19 `4fb3af93f` refactor + JSON-mode stdout flush fix; `320556dbf` markdown expanded view, chain streaming; 2025-12-23 `ce8a1c8eb` `cwd` param (#291).
- 2026-01-05 `c6fc08453` hooks + custom tools merged into extensions (#454); 2026-01-15 `ce7e73b50` YAML frontmatter parser (#728).
- 2026-02-08 `dae2eb5bf` list available agents in unknown-agent error (#1414); 2026-02-26 `7390f830d` `getAgentDir` for user agents (#1559).
- 2026-03-20 `74a46fc7e` file mutation queue for prompt temp writes; 2026-03-21 `7c92bb815` reuse current pi invocation (#2464).
- 2026-04-15 `462b3d21e` no `$bunfs` script path in child prompts (#3002).
- 2026-05-18 `93ecdbea3` full per-task parallel output (was 100-char previews) + failure diagnostics (#4710).
- 2026-08-11 `e3798ca91` inherit dispatcher model/thinking (#7897); 2026-08-14 `d268454e9` array-form `tools` (#7598); 2026-08-18 `8af7690c4` skip confirmation in trusted projects.
- 2026-09-22 `25cc5c7bf` README "Philosophy" manifesto (No sub-agents …) removed; stance survives in one README sentence (`de7e675de`).

## Evidence commits
`e9935beb5` `3424550d2` `eb1d08a5f` `4fb3af93f` `320556dbf` `ce8a1c8eb` `dae2eb5bf` `7390f830d` `74a46fc7e` `7c92bb815` `462b3d21e` `93ecdbea3` `e3798ca91` `d268454e9` `8af7690c4` `25cc5c7bf`

## Quirks
- Handles a `tool_result_end` event type that no package emits (grep `packages/*/src`: none) — dead branch; toolResult messages arrive via `message_end` anyway (`index.ts:384-387`).
- Parallel failure status check compares `stopReason !== "end"` — `"end"` is not a pi `StopReason` (`index.ts:673`; `types.ts:456`) (harmless).
- Child usage summed per subprocess; parent context only gets final text, not tool traces.
- `contextTokens` = last assistant `totalTokens`, not a running sum (`index.ts:375`).
- Tools are allowlisted per agent but "read-only" is prompt-only (reviewer.md:11) — no permission system ([[no-permission-prompts]]).
- Alternative substrate: `ctx.executeTool()` nested calls with `parentToolCallId` + bounded record (`docs/extensions.md:148`, `8562bcf66`) → [[nested-tool-calls]] / [[code-mode]] — tool-level orchestration without agent delegation.

## Failures
[[subagent-output-truncated]] · [[subagent-config-not-inherited]] · [[subagent-prompt-leaks-host-paths]]
