---
type: concept
stage: compaction
tier: candidate
aliases: [SUMMARIZATION_PROMPT, "context checkpoint", structured-summary-template, SUMMARY_TEMPLATE, compaction.txt, PROMPT_COMPACTION, "## Relevant Files"]
harnesses: [pi, opencode]
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
- **Fewer, merged sections** (opencode generation 3: Objective / Important Details / Work State Completed-Active-Blocked / Next Move / Relevant Files), every section kept with "(none)" when empty, "Do not mention the summary process".
- **Mandatory LLM-written Relevant Files section** instead of mechanical file lists (opencode `78f85b1cd6`) — cf. [[file-op-tracking]].
- **Language rule**: "Respond in the same language as the conversation" in the summarizer system prompt (opencode) → [[side-call-language-drift]].
- **Summarizer as a hidden agent** with all permissions denied and its own prompt file (opencode legacy `compaction` agent).

## Implementations
- [[pi--structured-compaction-summary|pi]] — `SUMMARIZATION_PROMPT` 6-section EXACT template since 2025-12-29; same skeleton for branch summaries; durable copy adds "fold an earlier summary in".
- [[opencode--structured-compaction-summary|opencode]] — shared `SUMMARY_TEMPLATE` (5 sections, third generation since 2026-07) + `compaction.txt` system prompt "Do not continue the conversation…"; v2 sends the template without a system prompt.

## Failures
- [[summary-template-drops-goals]]
- [[domain-biased-summarizer-prompt]]
- [[summarizer-refusal]]
- [[side-call-language-drift]]

## Tradeoffs
- [[compaction-design]]

## Related
[[iterative-summary-update]] · [[split-turn-summary]] · [[branch-summary]] · [[transcript-serialization-for-summary]] · [[summary-validation]] · [[file-op-tracking]] · [[auto-compaction]]
