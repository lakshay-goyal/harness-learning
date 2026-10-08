---
type: failure
concepts: [auto-compaction, xml-prompt-boundaries]
harnesses: [codex]
---
**Symptom** — After compaction the model received the summary as a bare user message; per the PR body, multiple user messages "confuses the model", and repeated compactions re-summarized previous summaries as if they were user requests.

**Root cause** — Summary text injected in the user role with no framing that both the model and the harness could recognize.

**Fix · [[codex]]** — `0b28e72b66` 2025-11-14 "Improve compact (#6692)": summary wrapped with `SUMMARY_PREFIX` ("Another language model started to solve this problem and produced a summary of its thinking process… use the information in this summary to assist with your own analysis:", `codex-rs/prompts/templates/compact/summary_prefix.md:1`); prefix-matched summaries filtered out of the user messages carried into the next compaction (`is_summary_message`, `codex-rs/core/src/compact.rs:583-603`); summary item tagged `compaction.summary` (`codex-rs/core/src/context/compaction_summary.rs:17-36`). PR body also flagged: "Theoretically, we can end up in infinite compaction loop if the user messages > compaction limit".

**Lesson** — Mark injected summaries with a recognizable framing so both the model and the next compaction can tell them apart from real user input.

Related: [[auto-compaction]] · [[xml-prompt-boundaries]] · [[message-role-layering]] · [[iterative-summary-update]] · [[codex--auto-compaction|codex]]
