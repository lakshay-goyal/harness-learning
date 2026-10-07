---
type: group
group: 05-context
---
Scope: what the model sees each request and how it is kept inside the window — app→wire message conversion, per-request context transforms, out-of-band message placement, tool-output bounding/spill, image normalization, token estimation, and the whole compaction family (trigger, cut point, summary prompts, validation, file tracking, background mode, branch summaries, overflow recovery).

## Concepts
### Request assembly
- [[message-conversion-layer]] — App-level message union converted to provider messages only at request boundary; custom message types.
- [[context-transform-hook]] — Per-request hook rewriting messages sent to the model without mutating persisted history.
- [[out-of-band-message-deferral]] — Buffer messages produced during a run (user shell, plugin notes) until turn boundary to keep tool-call adjacency.
- [[world-state-diff-injection]] — Typed environment/config sections diffed per request; only changed sections appended with replace/remove notices; snapshots persisted as merge patches.
- [[cross-session-memory]] — Background LLM pipeline distils past sessions into memory files; bounded summary injected into new sessions, details read on demand.
- [[synthetic-tool-call-injection]] — Work the user triggers directly, such as a `!` shell command or a subagent command, is recorded as an assistant tool call plus its result, so the model sees it in its own tool shape.
- [[project-references]] — Extra named directories outside the workspace advertised to the model (name/path/description) so it can read them on demand.
### Bounding inputs
- [[tool-output-truncation]] — Bound tool output by lines OR bytes (first hit), direction per tool (head/tail/middle), actionable continuation notice.
- [[tool-output-spill]] — Full output persisted to temp file; path named in result for later reads.
- [[image-normalization]] — Sniff, convert, resize, cap or block images before they enter history.
- [[token-estimation]] — Context size = last real provider usage + chars/N heuristic for newer messages.
- [[tool-output-pruning]] — Old tool-result bodies beyond a recent-token window are erased and replaced with a placeholder, as cheap relief before or instead of summarization.
### Compaction
- [[auto-compaction]] — Summarize old history when estimated tokens exceed window − reserve; keep recent window verbatim; summary re-injected as user message; manual /compact.
- [[compaction-cut-point]] — Rules for where history splits: never between tool call and result; valid cut entries.
- [[split-turn-summary]] — When one turn exceeds keep budget, summarize the turn prefix separately and merge.
- [[structured-compaction-summary]] — Fixed-section checkpoint template (Goal/Constraints/Progress/Decisions/Next/Critical context).
- [[iterative-summary-update]] — Feed previous summary into next compaction with merge rules.
- [[transcript-serialization-for-summary]] — Flatten conversation to tagged text + anti-continuation system prompt so summarizer doesn't continue it.
- [[summary-validation]] — Reject truncated, empty or tool-calling summaries before persisting.
- [[file-op-tracking]] — Carry cumulative read/modified file lists across compactions.
- [[background-compaction]] — Start summarizing below a soft threshold without blocking; apply at idle boundary.
- [[overflow-recovery]] — On overflow (explicit/silent/length) compact then retry exactly once.
- [[branch-summary]] — LLM summary of an abandoned session-tree branch injected where the new branch continues.
- [[model-requested-context-reset]] — Model sees remaining budget and can request a fresh window (no summary); continuity via model notes + searchable history.

## Neighbours
[[context-overflow-detection]] · [[max-tokens-context-clamp]] (02) · [[context-file-hierarchy]] · [[skill-progressive-disclosure]] · [[env-vars-as-context]] (04) · [[cache-retention-control]] (06) · [[context-projection]] · [[context-edit-overlay]] · [[session-tree]] (08) · [[session-handoff]] (09)

Failures: [[Context Failures]] · Axis: [[compaction-design]].
