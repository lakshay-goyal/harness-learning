---
type: concept
stage: compaction
tier: candidate
aliases: [getSummarizationFailure, "Summarization attempted to call a tool", summaryText, "generation hit the token cap", "(no summary available)", "Post-turn compaction completed without an assistant summary"]
harnesses: [pi, codex]
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
- **Phase-dependent strictness** ✔ codex: pre/mid-turn local compaction accepts an empty summary as "(no summary available)" and does not check length-truncation or tool calls; only post-turn (optional, at idle) requires a non-empty summary and commits buffered output only on success ([[no-summary-validation-local]]).
- **Structural validation of an opaque server summary**: exactly one compaction item, else fatal ✔ codex remote.

## Implementations
- [[pi--summary-validation|pi]] — `getSummarizationFailure` (error / length), tool-call rejection in 3 call sites, `prepareCompaction` returns undefined for empty ranges; durable requires clean stop + non-empty text.
- [[codex--summary-validation|codex]] — partial: empty → placeholder (pre/mid-turn), hard error (post-turn); remote checks "exactly one compaction item".

## Failures
- [[truncated-summary-persisted]]
- [[summarizer-emits-tool-calls]]
- [[empty-compaction-summary]]

## Related
[[auto-compaction]] · [[transcript-serialization-for-summary]] · [[branch-summary]] · [[split-turn-summary]] · [[errors-as-stream-events]] · [[truncated-tool-call-guard]] · [[compaction-locus]]
