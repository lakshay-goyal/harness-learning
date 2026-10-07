---
type: absence
harnesses: [codex]
---
# no-prompt-and-agent-hooks

Hook handler types `prompt` and `agent` (LLM-evaluated hooks) exist in the protocol/schema but are rejected at load; only command and MCP-tool handlers run.

**What's missing**
- Handler enum Command, McpTool, Prompt, Agent (`codex-rs/protocol/src/protocol.rs:1641-1646`), but discovery skips Prompt/Agent with "prompt hooks are not supported yet" / "agent hooks are not supported yet" (`codex-rs/hooks/src/engine/discovery.rs:637-656`).

**Evidence of decision**
- Schema-level placeholder for Claude-Code-style hooks (engine is `ClaudeHooksEngine`, `codex-rs/hooks/src/engine/dispatcher.rs:115`); no commit stating intent found (status: not yet, not never — unverified).

**Implication**
- LLM-judged gating must be done by a command hook calling a model itself, or by the built-in Guardian reviewer ([[llm-approval-reviewer]]).
- Imported Claude Code configs with prompt/agent hooks lose them ([[external-agent-import]]).

Related: [[turn-lifecycle-hooks]] · [[extension-event-hooks]] · [[codex--extension-event-hooks|codex]] · [[llm-approval-reviewer]] · [[Absences]]
