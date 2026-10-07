---
type: failure
concepts: [split-turn-summary, structured-compaction-summary, transcript-serialization-for-summary]
harnesses: [pi]
---
**Symptom** — Claude Fable 5.1 refused to produce split-turn (turn-prefix) compaction summaries, so compaction of long single turns failed.

**Root cause** — The prompt framed the task as "This is the PREFIX of a turn that was too large to keep. The SUFFIX (recent work) is being kept…" with the conversation in `<conversation>` XML tags — to a safety-tuned model this read like reconstructing a hidden/withheld part or an injection-shaped task.

**Fix · [[pi]]** — `d192bd6dc` 2026-09-22 (#9908, fixes #9652) "avoid Fable split-turn summary refusals": "Separate conversation content from instructions and replace prefix/suffix wording with continuation-oriented summarization guidance. Hopefully this avoids the refusals." Now `# Conversation` / `# Instructions` headings, "earlier context from an ongoing conversation. Later messages are stored separately and do not need to be reconstructed", "Create a concise checkpoint…", and "Only summarize information explicitly present above. Do not infer or recreate later messages." (`packages/coding-agent/src/core/compaction/compaction.ts:942-955`, `1096`). Full text in [[pi--split-turn-summary]].

**Lesson** — Summarization prompts are model-version-sensitive: phrase them as neutral checkpointing of what is present, separate data from instructions, avoid "reconstruct the hidden part" framing.

Related: [[split-turn-summary]] · [[structured-compaction-summary]] · [[transcript-serialization-for-summary]] · [[domain-biased-summarizer-prompt]] · [[server-side-refusal-fallback]]
