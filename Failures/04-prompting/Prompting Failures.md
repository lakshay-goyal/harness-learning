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
- [[edits-bypass-patch-tool]] — codex: ~22 % of edits via python/cat/sed; MUST → 0 % → reverted → carve-out wording.
- [[premature-turn-end]] — codex: stopped at analysis/proposal or re-asked permission; per-model "bias to action" sections.
- [[over-validation-in-interactive-mode]] — codex: ran slow tests/lint while a human waited; keyed off approval mode.

## Per-model prompt text (codex)
- [[formatting-instruction-echoed-literally]] — "Bold the keyword" copied into answers.
- [[output-breaks-client-renderer]] — fences without info string, unclickable file refs, ANSI codes.
- [[prompt-rewrite-drops-load-bearing-lines]] — GPT-5 rewrite dropped shell guidance; restored after evals.
- [[formatter-corrupts-prompt-file]] — prettier turned `*** Begin Patch` into `**_ Begin Patch`.
- [[anchor-based-prompt-injection-silently-fails]] — `replace("## Editing constraints", …)` no-op on prompts lacking the heading.

## Identity
- [[forced-model-identity-override]] — "You are actually not Claude, you are Pi." reverted after 11 days.

## Prompt structure & overrides
- [[markdown-boundaries-ingested-inconsistently]] — AGENTS.md headings blurred with prompt structure; cwd glued to appended text.
- [[forced-system-prompt-applied-as-late-update]] — forced prompt arrived as a later update; original prompt stayed leading.
- [[prompt-hook-chain-sees-stale-prompt]] — later `before_agent_start` handlers saw the base prompt, not earlier edits.
- [[windows-backslash-cwd-copied-into-shell]] — model pasted `C:\…` cwd into bash.
- [[mode-state-confusion]] — codex: Default vs Plan confusion; imperative user text treated as mode exit; question tool outside Plan.
- [[client-side-check-wrong-host]] — codex: `/init` existence check ran on the TUI host, not the remote executor.

## Context files, skills, docs
- [[context-file-loaded-twice-in-worktrees]] — AGENTS.md injected twice in nested git worktrees.
- [[context-file-discovery-filesystem-edge-cases]] — Windows walk hang; dirs named AGENTS.md → EISDIR; codex: path-syntax fallback names probed network paths.
- [[stale-context-files-mid-session]] — codex: global AGENTS.md edits ignored until restart; now replacement notices per request.
- [[self-reference-not-routed-to-docs]] — codex: "you/your/this app" questions didn't trigger the docs skill.
- [[skills-hidden-when-read-tool-absent]] — skills vanished with bash-only toolsets; hint named hidden reader.
- [[instruction-relative-paths-resolved-from-cwd]] — skill/doc relative paths resolved against the user's cwd.

## See also (primary concept in other groups)
- [[foreign-harness-tool-hallucination]] — Codex models called `apply_patch`/`update_plan` (bridge prompt saga) — [[Tools Failures]].
- [[partial-file-read-acted-on]] — acted on first 2000 lines of files/pi docs — [[Tools Failures]].
- [[tool-description-lies-about-async]] — tool description as executable spec — [[Tools Failures]].
- [[edit-tool-dual-mode-confusion]] — two schema shapes = prompt bug — [[Tools Failures]].
- [[placeholder-text-misleads-model]] — placeholder strings are prompts — [[Model Interface Failures]].
- [[volatile-system-prompt-prefix]] — date in the prompt busted caches — [[Caching Failures]].
- [[tool-loadout-stale-within-run]] — run prompt dropped on tool refresh — [[Loop Failures]].
- [[prompt-states-stale-harness-limits]] — codex prompt said "10 kilobytes or 256 lines" after limits changed — [[Context Failures]].
- [[destructive-git-on-user-changes]], [[approval-asked-in-prose]], [[approval-circumvention]], [[sandbox-network-error-not-escalated]], [[model-proposed-rule-too-broad]] — codex permission-prompt failures — [[Safety Failures]].
- [[time-pressure-shortcuts]], [[interrupted-turn-invisible-to-model]] — wording of injected loop text — [[Loop Failures]].
- [[summarizer-refusal]], [[summarizer-continues-conversation]], [[summary-template-drops-goals]], [[compaction-request-shape-mismatch]], [[truncated-summary-persisted]] — compaction/summary prompt failures — [[Context Failures]].
