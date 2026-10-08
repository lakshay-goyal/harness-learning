---
type: concept
stage: context
tier: variant
aliases: [WorldState, WorldStateSection, WorldStateSnapshot, WorldStateItem, render_history_diff, render_full, merge_patch_from, reference_context_item, build_world_state_for_step, record_step_world_state_if_changed, "Placement::Standalone", environment-context-diffs]
harnesses: [codex]
---
Model-visible environment/config state is kept as typed sections with comparison snapshots; before each model request only sections whose snapshot changed are rendered and appended to history (with explicit "replaces"/"no longer applies" notices), and snapshots are persisted as merge patches so resume/fork keep diffing.

## Why
- Settings change mid-session (cwd/environments, permissions, AGENTS.md edits, mode, model, tools); rewriting the system prompt busts the cache, never telling the model leaves it acting on stale state ([[stale-context-files-mid-session]]).
- Re-sending whole blocks on every change bloats context ([[permission-context-reinjected-repeatedly]]).
- "Initial injection, turn-to-turn updates, and changes that happen within a turn use different baselines" (`3b32d861c5`) — one diff engine replaces three ad-hoc paths.
- Resume/fork must know what the model was last told, or it re-injects everything / nothing.

## Design space
- Static prompt snapshot at session start, never updated ✔ codex before 2026-06 for global AGENTS.md ([[stale-context-files-mid-session]]).
- Re-render the system prompt per request (cache cost) vs **append deltas, never rewrite prefix** ✔ codex, ✔ pi (section patches, [[transcript-carried-system-prompt]]).
- **Typed sections with persisted IDs and snapshot comparison (`Absent | Unknown | Known`)** ✔ codex.
- Change wording: explicit replacement/removal sentences per section ✔ codex.
- Granularity: per sampling request (incl. mid-turn) ✔ codex vs per user turn.
- Persistence: full snapshot baseline + RFC 7386 merge patches in the transcript ✔ codex vs re-derive from log per request (pi [[context-projection]]).
- After compaction: re-render all sections in full and insert before the last real user message ✔ codex.
- Incremental deltas for append-only data (approved command prefixes, network rules, tool catalog) instead of re-sending the block ✔ codex.

## Implementations
- [[codex--world-state-diff-injection|codex]] — `codex-rs/core/src/context/world_state/*` sections (agents_md, base_instructions, permissions, collaboration_mode, environments, tools, top_level_tools, multi_agent_mode, context_window_guidance…), diff recorded before every sampling request, `RolloutItem::WorldState{full, state}` merge patches.

## Failures
- [[stale-context-files-mid-session]] (04-prompting)
- Cross-group: [[permission-context-reinjected-repeatedly]] (06-caching) · [[tool-loadout-stale-within-run]] (01-loop) · [[fork-carries-startup-context]] (08-state)

## Related
[[transcript-carried-system-prompt]] · [[message-role-layering]] · [[context-projection]] · [[cache-stable-prompt-prefix]] · [[context-file-hierarchy]] · [[plan-mode]] · [[current-time-reminder]] · [[auto-compaction]] · [[cache-preserving-config-update]]
