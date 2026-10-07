---
type: concept
stage: tools
tier: candidate
aliases: [createErrorToolResult, isError, tool-errors-as-results, "Tool X not found", "Operation aborted"]
harnesses: [pi]
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

## Implementations
- [[pi--tool-error-as-result|pi]] — `createErrorToolResult` on every failure path in `prepareToolCall`/`executeToolCall`/`finalize`; `runToolCall` never rejects; bash/MCP return isError with structured data.

## Failures
- [[bash-spawn-errors-crash-session]]
- [[hook-throw-aborts-parallel-batch]]
- [[signal-killed-command-reported-success]]

## Related
[[turn-loop]] · [[parallel-tool-execution]] · [[tool-call-gate]] · [[tool-result-rewriting]] · [[truncated-tool-call-guard]] · [[transcript-replay-repair]] · [[structured-tool-output]] · [[errors-as-stream-events]]
