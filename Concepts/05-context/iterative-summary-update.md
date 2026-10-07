---
type: concept
stage: compaction
tier: candidate
aliases: [UPDATE_SUMMARIZATION_PROMPT, UPDATE_SUMMARIZATION_INSTRUCTIONS, "<previous-summary>", previousSummary]
harnesses: [pi]
---
On the Nth compaction, the previous summary is passed alongside only the *new* messages with explicit merge rules (preserve, add, move In Progress → Done, update Next Steps), instead of summarizing a transcript that contains the old summary as just another message.

## Why
- Re-summarizing a summary each time compounds loss ("summary of a summary"); explicit PRESERVE rules slow the decay.
- The new-message range must start exactly where the previous summary's coverage ended — i.e. at the previous compaction's *first kept entry*, not at the compaction marker — or messages kept last time vanish from both context and summary (pi: [[repeated-compaction-drops-kept-messages]]).

## Design space
- **Separate update prompt + `<previous-summary>` block** ✔ pi coding-agent (`09d6131be`).
- **Single prompt; old summary is the head of the serialized range** with "If the conversation starts with an earlier summary, preserve its information and fold the newer messages into it" ✔ pi durable.
- No special handling (summary is ordinary context, re-summarized).
- Permission to drop: pi allows "If something is no longer relevant, you may remove it" — trades bounded size against loss.
- Range start: previous compaction marker (buggy) vs previous `firstKept` ✔ pi.
- Split-turn interplay: if nothing new to summarize, pi reuses `previousSummary` verbatim ("No prior history." if none) and only adds the turn prefix.

## Implementations
- [[pi--iterative-summary-update|pi]] — `UPDATE_SUMMARIZATION_PROMPT` with 6 merge rules + section hints "(preserve all previous, add new)"; previous summary passed in `<previous-summary>` after `<conversation>`; range starts at previous kept entries.

## Failures
- [[repeated-compaction-drops-kept-messages]]

## Related
[[structured-compaction-summary]] · [[auto-compaction]] · [[compaction-cut-point]] · [[file-op-tracking]] · [[transcript-serialization-for-summary]]
