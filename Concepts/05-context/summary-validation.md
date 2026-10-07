---
type: concept
stage: compaction
tier: candidate
aliases: [getSummarizationFailure, "Summarization attempted to call a tool", summaryText, "generation hit the token cap"]
harnesses: [pi, opencode]
---
A summary replaces history, so it is validated like a write to disk before persisting: reject error/aborted, length-truncated, tool-calling and empty outputs, and refuse to start when there is nothing to summarize.

## Why
- A length-stopped summary is silently partial; once persisted as the checkpoint, the lost part is gone from the model's view forever (pi: [[truncated-summary-persisted]]).
- Summarizers sometimes emit tool calls instead of text ([[summarizer-emits-tool-calls]]).
- Empty ranges produced empty "compactions" that showed start/end UI with nothing done ([[empty-compaction-summary]]).

## Design space
- Accept only clean `stop` with non-empty text and no tool call ✔ pi durable (`summaryText`).
- Reject `error` + `length`, reject tool calls; empty text not rejected ✔ pi coding-agent (`getSummarizationFailure`; empty text would persist — inferred from code).
- Prevent at source vs validate after: pi tried `toolChoice:"none"` (prevent) then dropped it for portability and kept post-hoc rejection.
- Pre-flight: return "nothing to compact" before emitting start events ✔ pi.
- On failure: keep history untouched, surface error event, no internal retry for deterministic failures ✔ pi; transient errors retried with the session retry policy.
- Branch summary failures return `{error}` instead of throwing; abort → `{aborted:true}`.
- **Abort mid-stream on a tool call**: processor throws "Tool call not allowed while generating summary" as soon as tool input starts (opencode legacy).
- **Fit pre-check**: skip when the summary prompt estimate exceeds `context − summaryOutput` (opencode v2).
- **Durable cutover only on a validated end event**: `Compaction.Started` may exist without `Ended`; only `Ended` activates the boundary (opencode v2).

## Implementations
- [[pi--summary-validation|pi]] — `getSummarizationFailure` (error / length), tool-call rejection in 3 call sites, `prepareCompaction` returns undefined for empty ranges; durable requires clean stop + non-empty text.
- [[opencode--summary-validation|opencode]] — legacy: `tools: {}` + throw on summary tool calls, compaction overflow terminal, no empty/length check; v2: empty/error rejection, prompt-fit pre-check, `Compaction.Ended` gate.

## Failures
- [[truncated-summary-persisted]]
- [[summarizer-emits-tool-calls]]
- [[empty-compaction-summary]]

## Related
[[auto-compaction]] · [[transcript-serialization-for-summary]] · [[branch-summary]] · [[split-turn-summary]] · [[errors-as-stream-events]] · [[truncated-tool-call-guard]]
