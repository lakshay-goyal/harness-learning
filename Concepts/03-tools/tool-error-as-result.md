---
type: concept
stage: tools
tier: must-have
aliases: [createErrorToolResult, isError, tool-errors-as-results, "Tool X not found", "Operation aborted", failUnsettledTools, "Tool execution failed", FunctionCallError::RespondToModel, FunctionCallError::Fatal]
harnesses: [pi, opencode, codex]
---
Every tool failure — unknown tool, invalid arguments, blocked by policy, aborted, thrown during execution or in a post-hook — becomes an `isError` tool result fed back to the model, never an exception that escapes the loop.

## Why
- An uncaught error inside a tool is a harness crash (pi: missing cwd → uncaught ENOENT killed the session, [[bash-spawn-errors-crash-session]]).
- In a parallel batch, one exception must not abort siblings ([[hook-throw-aborts-parallel-batch]]).
- Providers require a result for every call; a missing result breaks history replay ([[transcript-replay-repair]]).
- The error text is the model's only recovery signal, so it must be actionable and true ([[signal-killed-command-reported-success]]).

## Design space
- Tools **throw**, loop renders the message as result (pi final design after a same-day flip-flop in Nov 2025).
- Tools **return** error content (pi tried `f147109da`, reverted within hours).
- Hybrid: return `isError` with data when the failure carries payload (pi: bash non-zero exit, MCP isError) so programmatic callers keep structured output.
- Fail-closed policy hooks: hook throw = block (pi) vs fail-open.
- Expected failures as typed `Result` values at the I/O layer (pi durable `ExecutionEnv`).
- Crash-interrupted calls get a synthetic "interrupted" error result (pi durable) → [[crash-safe-tool-replay]].
- Two classes: model-visible error result vs **Fatal** that ends the turn for host-side failures where a model retry is pointless (✔ codex: clock, sleep, user-input channel, task join).
- Error output keeps the call's wire shape — custom-tool output, empty `tool_search` output, function output (✔ codex `failure_response`) → [[tool-wire-kinds]].
- Discovery failure returns an empty result, not an error (✔ codex `tool_search`).
- Synthesized paired output for aborted / output-less calls ("aborted by user after Xs", "aborted" at prompt build) (✔ codex).
- Unexpected tool defects returned to the model and the loop continues (opencode v2 `failUnsettledTools`).
- Permission rejection feedback as model-facing text (opencode legacy) vs dropped into a generic error (opencode v2, latent) → [[permission-ruleset]].

## Implementations
- [[pi--tool-error-as-result|pi]] — `createErrorToolResult` on every failure path in `prepareToolCall`/`executeToolCall`/`finalize`; `runToolCall` never rejects; bash/MCP return isError with structured data.
- [[codex--tool-error-as-result|codex]] — `RespondToModel` vs `Fatal`; wire-shaped failure items; "unsupported call: {name}"; sandbox denial as normal output.
- [[opencode--tool-error-as-result|opencode]] — tools die → SDK `tool-error` → error part; unknown names → `invalid` sink; one edit error per remedy; v2 fails unsettled calls with text and continues; v2 leaves collapse permission feedback into generic "Unable to …".

## Failures
- [[spill-failure-reports-successful-side-effect-as-failed]]
- [[subagent-error-reported-as-success]]
- [[permission-feedback-lost-in-tool-error]]
- [[bash-spawn-errors-crash-session]]
- [[hook-throw-aborts-parallel-batch]]
- [[signal-killed-command-reported-success]]
- [[ambiguous-tool-error-causes-retry-loop]]
- [[mcp-error-result-treated-as-success]]

## Related
[[turn-loop]] · [[parallel-tool-execution]] · [[tool-call-gate]] · [[tool-result-rewriting]] · [[truncated-tool-call-guard]] · [[transcript-replay-repair]] · [[structured-tool-output]] · [[errors-as-stream-events]] · [[tool-wire-kinds]] · [[abort-propagation]]
