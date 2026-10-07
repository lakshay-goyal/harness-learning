---
type: concept
stage: compaction
tier: candidate
aliases: [SUMMARIZATION_PROMPT, "context checkpoint", structured-summary-template]
harnesses: [pi]
---
Summaries follow a fixed-section checkpoint template (Goal / Constraints & Preferences / Progress Done-InProgress-Blocked / Key Decisions / Next Steps / Critical Context) that preserves exact paths, identifiers and error strings.

## Why
- Free-form "summarize the conversation" output drifts between compactions, drops goals of multi-task sessions and loses exact identifiers the next turn needs (pi's v1 free-form prompt replaced within 4 weeks, `ac71aac09`).
- A stable section skeleton makes the *next* compaction mergeable (move In Progress → Done) — the basis of [[iterative-summary-update]].
- Over-tight section constraints lose information: "Goal: 1-2 sentences" dropped goals in multi-task sessions ([[summary-template-drops-goals]]).

## Design space
- Free-form handoff note (pi v1 `6c2360af2`, OpenCode, Claude Code style) vs **EXACT fixed sections** ✔ pi.
- Sections: pi = Goal, Constraints & Preferences, Progress (Done / In Progress / Blocked as checkboxes), Key Decisions (`**Decision**: rationale`), Next Steps (ordered), Critical Context; branch summary drops Critical Context.
- Explicit preservation clause ("Preserve exact file paths, function names, and error messages") ✔ pi.
- Domain neutrality: "AI coding assistant" → "AI assistant" ✔ pi (`72fd91135`) so non-coding agents built on the harness aren't biased.
- Machine-appended structured data outside the LLM text (pi appends `<read-files>`/`<modified-files>`) → [[file-op-tracking]].
- Template replaceable by user? pi: compaction only appends "Additional focus"; branch summary allows full `replaceInstructions`; plugins can replace the whole summarizer.

## Implementations
- [[pi--structured-compaction-summary|pi]] — `SUMMARIZATION_PROMPT` 6-section EXACT template since 2025-12-29; same skeleton for branch summaries; durable copy adds "fold an earlier summary in".

## Failures
- [[summary-template-drops-goals]]
- [[domain-biased-summarizer-prompt]]
- [[summarizer-refusal]]

## Related
[[iterative-summary-update]] · [[split-turn-summary]] · [[branch-summary]] · [[transcript-serialization-for-summary]] · [[summary-validation]] · [[file-op-tracking]] · [[auto-compaction]]
