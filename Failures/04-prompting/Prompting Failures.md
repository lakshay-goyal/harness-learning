---
type: group
group: 04-prompting
---
Failures whose primary concept is in [[Prompting]].

## Standing rules & wording
- [[model-echoes-work-via-shell]] — model `cat`/echoed its summary through bash instead of answering in text.
- [[shell-cat-instead-of-read-tool]] — model read files with `cat`/`sed` instead of the read tool.
- [[imperative-guideline-over-compliance]] — "Inspect PI_* …" made models run env inspections every turn.
- [[prompt-names-unavailable-tools]] — rules named/implied tools the request didn't declare (grep/find/ls preference, READ-ONLY mode, hidden/codemode tools).
- [[over-commenting-code]] — models over-comment; rules diverge per family, origin/v2 keeps a density rule for Claude (opencode).
- [[autonomy-prompt-overreach]] — Codex-style persistence push without scope bound; replaced on origin/v2 (opencode).

## Per-model prompts
- [[borrowed-prompt-foreign-references]] — prompts lifted from gemini-cli/Copilot/Claude Code/Codex reference things opencode lacks (opencode).
- [[client-unrenderable-output-format]] — Codex-desktop link format rendered badly in the TUI (opencode).
- [[excessive-permission-questions]] — codex models asked "Should I proceed?" (opencode).
- [[model-cannot-parallel-tool-call]] — Trinity needs one tool per message (opencode).

## Identity
- [[forced-model-identity-override]] — "You are actually not Claude, you are Pi." reverted after 11 days.

## Prompt structure & overrides
- [[markdown-boundaries-ingested-inconsistently]] — AGENTS.md headings blurred with prompt structure; cwd glued to appended text.
- [[forced-system-prompt-applied-as-late-update]] — forced prompt arrived as a later update; original prompt stayed leading.
- [[prompt-hook-chain-sees-stale-prompt]] — later `before_agent_start` handlers saw the base prompt, not earlier edits.
- [[windows-backslash-cwd-copied-into-shell]] — model pasted `C:\…` cwd into bash.

## Context files, skills, docs
- [[context-file-loaded-twice-in-worktrees]] — AGENTS.md injected twice in nested git worktrees.
- [[context-file-discovery-filesystem-edge-cases]] — Windows walk hang; dirs named AGENTS.md → EISDIR.
- [[skills-hidden-when-read-tool-absent]] — skills vanished with bash-only toolsets; hint named hidden reader.
- [[instruction-relative-paths-resolved-from-cwd]] — skill/doc relative paths resolved against the user's cwd.
- [[duplicated-catalog-in-prompt]] — skill catalog rendered in system prompt, tool description and kimi.txt (opencode).
- [[generic-context-file-bloat]] — `/init` produced 150-line generic AGENTS.md files (opencode).

## See also (primary concept in other groups)
- [[foreign-harness-tool-hallucination]] — Codex models called `apply_patch`/`update_plan` (bridge prompt saga) — [[Tools Failures]].
- [[partial-file-read-acted-on]] — acted on first 2000 lines of files/pi docs — [[Tools Failures]].
- [[tool-description-lies-about-async]] — tool description as executable spec — [[Tools Failures]].
- [[edit-tool-dual-mode-confusion]] — two schema shapes = prompt bug — [[Tools Failures]].
- [[tool-description-drifts-from-implementation]] — borrowed/stale tool descriptions (opencode) — [[Tools Failures]].
- [[todo-tool-usage-calibration]] — todo instructions per model family (opencode) — [[Tools Failures]].
- [[edit-oldstring-drops-lines]] — Meta prompt diff-before-edit rule (opencode) — [[Tools Failures]].
- [[placeholder-text-misleads-model]] — placeholder strings are prompts — [[Model Interface Failures]].
- [[volatile-system-prompt-prefix]] — date in the prompt busted caches — [[Caching Failures]].
- [[tool-loadout-stale-within-run]] — run prompt dropped on tool refresh — [[Loop Failures]].
- [[summarizer-refusal]], [[summarizer-continues-conversation]], [[summary-template-drops-goals]], [[compaction-request-shape-mismatch]], [[truncated-summary-persisted]] — compaction/summary prompt failures — [[Context Failures]].
