---
type: implementation
harness: codex
concept: tool-safety-annotations
commit: 622e9e3696
files: [codex-rs/core/src/mcp_tool_call.rs:2461, codex-rs/core/src/mcp_tool_call.rs:2480, codex-rs/connectors/src/app_tool_policy.rs:235, codex-rs/protocol/src/protocol.rs:4194, codex-rs/core/src/guardian/approval_request.rs:126, codex-rs/core/src/tools/handlers/mcp.rs:142]
---
[[tool-safety-annotations]] in [[codex]].

## Mechanism
- MCP approval in `Auto` mode (`requires_mcp_tool_approval`, `codex-rs/core/src/mcp_tool_call.rs:2461-2478`): `destructiveHint == true` → approval (short-circuits even if `readOnlyHint` is set); else `readOnlyHint == true` → no approval; else approval if destructive (default **true** when missing) or open-world (default **true** when missing). Missing annotations ⇒ prompt.
- Per-tool override `AppToolApproval::{Auto, Prompt, Writes, Approve}`: Prompt = always, Writes = unless read-only, Approve = never (`codex-rs/core/src/mcp_tool_call.rs:2480-2492`). Connectors use the same defaults (`codex-rs/connectors/src/app_tool_policy.rs:235-236`).
- Persistence: `ReviewDecision::ApprovedMcpPolicyAmendment` = cross-session MCP allow (`codex-rs/protocol/src/protocol.rs:4194-4196`). Granular `mcp_elicitations` flag covers MCP elicitation prompts ([[codex--approval-policy-modes]]).
- Reviewer input: guardian receives the hints with the MCP request (`codex-rs/core/src/guardian/approval_request.rs:126-130`); descriptions shown as untrusted ([[codex--llm-approval-reviewer]]).
- Concurrency signal: `readOnlyHint: true` ⇒ MCP call parallel-safe (`c83ba22359`); stale read-only hints cleared on cached tools before live startup (`codex-rs/core/src/tools/handlers/mcp.rs:142-150`; `3bbf1fe757`) — see [[unannotated-mcp-tools-serialized]].
- Built-in tools carry no MCP-style hints; their risk comes from sandbox + approval policy.

## Evolution
- 2026-02-20 `d3cf8bd0fa` "require approval for destructive MCP tool calls (#12353)".
- 2026-03-25 `32c4993c8a` "default approval behavior for mcp missing annotations (#15519)" — spec-pessimistic defaults.
- 2026-05-21 `c83ba22359` read-only hint as parallel signal.

## Versus pi
- pi exposes the same MCP hints to extensions only, no built-in consumer, MCP defaults followed in docs ([[pi--tool-safety-annotations]]). Codex consumes them in core to decide approvals and parallelism.
