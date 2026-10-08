---
type: failure
concepts: [context-file-hierarchy, transcript-carried-system-prompt, world-state-diff-injection]
harnesses: [codex]
---
**Symptom** — "Running root threads retained their startup global instructions, so edits to global `AGENTS.md` files did not take effect during an active session." (`935ac7710d` body); cached instructions could also survive a later permission tightening (`7ece061767`).

**Root cause** — Context files were snapshotted once at session start; changing them by rewriting the prompt would bust the cache, so nothing updated them.

**Fix · [[codex]]** — `f2f80ef442` 2026-06-24 replacement/removal notices; `935ac7710d` 2026-09-10 reload global AGENTS.md at every model-request boundary (even between tools in one turn) and append "These AGENTS.md instructions replace all previously provided AGENTS.md instructions." / "The previously provided AGENTS.md instructions no longer apply." (`codex-rs/core/src/context/world_state/agents_md.rs:11-13`, `:44-80`); `7ece061767` 2026-08-20 enforce the filesystem sandbox on AGENTS.md reads.

**Lesson** — When context files can change mid-session, update by appended replacement notices (cache-safe), not by rewriting history or ignoring the change.

Related: [[context-file-hierarchy]] · [[transcript-carried-system-prompt]] · [[world-state-diff-injection]] · [[codex--context-file-hierarchy|codex]]
