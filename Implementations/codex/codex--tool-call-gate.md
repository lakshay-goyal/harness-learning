---
type: implementation
harness: codex
concept: tool-call-gate
commit: 622e9e3696
files: [codex-rs/core/src/tools/registry.rs:609, codex-rs/core/src/tools/registry.rs:626, codex-rs/core/src/tools/registry.rs:747, codex-rs/hooks/src/events/pre_tool_use.rs:200, codex-rs/hooks/src/events/permission_request.rs:1, codex-rs/core/src/tools/approvals.rs:505, codex-rs/core/src/tools/orchestrator.rs:1, codex-rs/core/src/stream_events_utils.rs:400]
---
[[tool-call-gate]] in [[codex]].

## Mechanism
Codex has three stacked gates on every tool call, all in core:
1. **PreToolUse hooks** (external processes, Claude-Code-compatible protocol) in `ToolRegistry::dispatch_any_with_state` (`codex-rs/core/src/tools/registry.rs:626-678`): can block (→ `RespondToModel(message)`, the model sees it as the tool output) or rewrite input (`updated_input`) before the handler; hook wait cancellable. PostToolUse after the handler (`:747`). Payload/kind mismatch is `Fatal` and ends the turn (`:609-624`). Router refusals become a `function_call_output` and force a follow-up (`codex-rs/core/src/stream_events_utils.rs:400-425`).
   - Block = exit code 2 with stderr reason or JSON block decision; **any other failure — spawn error, non-0/2 exit, invalid JSON, exit 2 without reason — marks the hook `Failed` but does not block** (`codex-rs/hooks/src/events/pre_tool_use.rs:200-288`) → fail-open by protocol ([[hook-error-fails-open]]). Multiple hooks: any block wins (`:115`).
2. **Approval orchestration** `ToolOrchestrator`: approval → select sandbox → attempt → escalate (`codex-rs/core/src/tools/orchestrator.rs:1-7`), driven by [[approval-policy-modes]], [[command-rule-policy]], [[dangerous-command-heuristics]], [[tool-safety-annotations]] (MCP).
   - Approval precedence: PermissionRequest hooks (allow/deny/no decision; "Unlike `pre_tool_use`, handlers do not rewrite tool input or block by stopping execution outright", `codex-rs/hooks/src/events/permission_request.rs:1-6`; a failed hook → no decision → normal flow, i.e. still asks) → guardian → user (`codex-rs/core/src/tools/approvals.rs:505-525`).
3. **OS sandbox** as the backstop ([[os-level-sandbox]]).
- Nested coverage: per-`execve` gating inside scripts via [[per-exec-interception]]; code-mode and MCP calls go through the same registry dispatch (unverified beyond registry entry point).
- Approval keyed by call/request id (`c4b771a16f`, [[approval-scope-too-broad-in-parallel-batch]]); on interrupt, task cancelled before pending approvals are cleared (`codex-rs/core/src/tasks/mod.rs:569-572`, `:620-622`; `ad57505ef5`, [[approval-wait-surfaces-as-rejection-on-interrupt]]).

## Evolution
- 2025-04 TS approval modes (suggest / auto-edit / full-auto) → 2025-04-24 `58f0e5ab74` execpolicy crate → 2025-11-17 `a941ae7632` execpolicy v2 (legacy removed `656a2d0905` 2026-07-10).
- 2025-10-20 `5e4f3bbb0b` orchestrator ("rework tools execution workflow").
- 2025-12-10 `c4af707e09` experimental "command risk assessment" removed.
- 2026-03-07 `e84ee33cc0` guardian LLM reviewer; 2026-05-12 `d996f5366f` guardian as extension; 2026-08-13 `fe614a6304` Guardian V2; 2026-08-19 `e741cd9ace` consolidated.
- 2026-05-05 `0452dca986` hook trust (unmanaged hooks must be reviewed before they run).

## Versus pi
- pi: one in-process `beforeToolCall` seam, fail-closed on handler throw, no built-in policy ([[pi--tool-call-gate]]). Codex: out-of-process hooks that fail **open**, plus built-in approval policy, rules, reviewer and kernel sandbox — policy lives in core, hooks are an extra.
