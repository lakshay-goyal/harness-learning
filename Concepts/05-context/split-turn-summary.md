---
type: concept
stage: compaction
tier: candidate
aliases: [isSplitTurn, turnPrefixMessages, TURN_PREFIX_SUMMARIZATION_PROMPT, "Turn Context (split turn)", split-turn-compaction]
harnesses: [pi]
---
When a single user turn (request + long tool loop) is larger than the keep budget, the cut lands inside it; the turn's prefix is summarized separately with a turn-focused prompt and appended to the history summary.

## Why
- In agentic coding one "turn" can be dozens of tool calls; cutting only at turn starts would keep the whole turn (no room) or summarize it wholesale (losing the original request mid-task).
- Without the prefix summary the kept suffix starts with tool calls/results whose purpose (the user's request) is gone.
- The prompt wording is model-sensitive: pi's "PREFIX/SUFFIX of a turn that was too large to keep" framing triggered refusals from a safety-tuned model ([[summarizer-refusal]]).
- Fanning out the two summary calls in parallel broke single-slot local providers ([[parallel-side-requests-single-slot-provider]]).

## Design space
- **Split allowed** ✔ pi coding-agent vs never split (pi durable: no split-turn prefix summary, cut at any user/assistant entry and the prefix is simply part of the summarized range).
- **Prefix prompt**: dedicated short template (Original Request / Progress So Far / Context Needed to Continue) ✔ pi vs reuse main template.
- **Budget**: smaller output cap for prefix (pi 0.5×reserve = 8192) vs same.
- **Merge format**: concatenate `history --- **Turn Context (split turn):** prefix` ✔ pi vs one combined call (pi v1 `5a9d844f9` "Merge turn prefix summary into main summary", later split again).
- **Call scheduling**: parallel (`Promise.all`) → **sequential** ✔ pi (`f58c11562`).
- **Framing**: XML `<conversation>` + prefix/suffix jargon (refused) → `# Conversation` / `# Instructions` headings + "Do not infer or recreate later messages" ✔ pi (`d192bd6dc`).

## Implementations
- [[pi--split-turn-summary|pi]] — cut-point marks `isSplitTurn`; history summary and turn-prefix summary generated sequentially, merged with a `---` separator; prefix prompt rewritten 2026-09-22 after Fable refusals.

## Failures
- [[summarizer-refusal]]
- [[parallel-side-requests-single-slot-provider]]

## Related
[[compaction-cut-point]] · [[auto-compaction]] · [[structured-compaction-summary]] · [[transcript-serialization-for-summary]] · [[summary-validation]]
