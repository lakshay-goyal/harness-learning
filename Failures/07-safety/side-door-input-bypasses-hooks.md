---
type: failure
concepts: [tool-call-gate, extension-event-hooks, tool-result-rewriting]
harnesses: [pi, opencode]
---
**Symptom** — Inputs and tool executions reached the agent through paths that skipped the extension hooks used as the policy layer: RPC `steer`/`follow_up` skipped `input` handlers (and skill/template expansion); RPC `bash` skipped `user_bash`; custom tools were not wrapped by `tool_call`/`tool_result` hooks; `tool_result` hooks never saw thrown errors (and always saw `isError:false`); error-result overrides from `afterToolCall` were dropped; multiple `tool_result` handlers were last-wins, losing earlier patches.

**Root cause** — Interception was attached per entry point / per tool wrapper instead of at one choke point; every new input path (RPC commands, queued messages, custom tools, error branch) had to remember to call the hooks.

**Fix · [[pi]]**
- 0.24.1 (#248) — hooks wrap custom tools (`packages/coding-agent/CHANGELOG.md:5501`).
- 0.31.0 (#374) — `tool_result` emitted on thrown errors with correct `isError` (`packages/coding-agent/CHANGELOG.md:5214`).
- `2668326e0` 2026-02-06 (#1280) — `tool_result` patches chain instead of last-wins.
- `63ac2df24` 2026-03-14 (#2113) — interception moved from wrappers into agent-core `beforeToolCall`/`afterToolCall` (`packages/coding-agent/src/core/agent-session.ts:652-724`) — one seam for all tools incl. nested codemode/MCP calls.
- `e9808b585` 2026-04-17 (#3051) — forward `details`/`isError` overrides for error results (`packages/coding-agent/CHANGELOG.md:2458`).
- `5d548ae96` 2026-07-28 (#7214) — RPC bash runs `user_bash`.
- `faa9863cb` 2026-09-08 (#8718) — queued steer/follow-up messages run `input` handlers + expansion (`packages/coding-agent/src/core/agent-session.ts:2172-2199`).

**Fix · [[opencode]]** — open (observed at `ecc4916b5a`). Command templates expand `` !`cmd` `` by running the preferred shell directly (`packages/opencode/src/session/prompt.ts:1397-1408`; regex `packages/opencode/src/config/markdown.ts:6`), outside the `bash` permission, the tree-sitter scan and the `tool.execute.before` hook. User arguments are substituted before expansion (`packages/opencode/src/session/prompt.ts:1383-1395`), and MCP prompts are surfaced as command templates (`packages/opencode/src/command/index.ts:105-125`), so an MCP-supplied prompt containing `` !`…` `` would also run when invoked (inferred, not reproduced).

**Lesson** — If hooks are your permission layer, every input path and every result branch must funnel through one choke point; audit new entry points (RPC, queues, nested calls) against it.

Related: [[tool-call-gate]] · [[extension-event-hooks]] · [[tool-result-rewriting]] · [[headless-rpc-mode]] · [[steering-queue]] · [[pi--tool-call-gate|pi]] · [[hook-error-fails-open]] · [[opencode--tool-call-gate|opencode]] · [[shell-command-permission-parsing]]
