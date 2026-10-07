---
type: implementation
harness: codex
concept: world-state-diff-injection
commit: 622e9e3696
files: [codex-rs/core/src/context/world_state/mod.rs:301, codex-rs/core/src/context/world_state/mod.rs:412, codex-rs/core/src/session/world_state.rs:40, codex-rs/core/src/session/world_state.rs:146, codex-rs/core/src/session/mod.rs:3614, codex-rs/core/src/session/turn.rs:504, codex-rs/core/src/session/turn.rs:1448, codex-rs/protocol/src/protocol.rs:3330, codex-rs/core/src/context/world_state/agents_md.rs:11, codex-rs/core/src/context/world_state/base_instructions.rs:12, codex-rs/core/src/context_manager/updates.rs:1]
---
[[world-state-diff-injection]] in [[codex]].

## Mechanism
- **Sections**: typed `WorldStateSection`s with a stable persisted `ID` — agents_md, base_instructions, model, model_catalog, permissions, approved_command_prefixes, collaboration_mode, persistent_mode, environments, environments_instructions, apps/plugins instructions, tools, top_level_tools, multi_agent_mode, realtime, context_window_guidance, managed_developer_instructions, token budget context; extensions add their own (skills: `skills`, `cloud_skills`, `host_skills`, `codex-rs/ext/skills/src/world_state.rs:12-24`) (`const ID` lines under `codex-rs/core/src/context/world_state/`, e.g. `agents_md.rs:37`; assembly `codex-rs/core/src/session/world_state.rs:146-399`).
- **Trait contract** (`codex-rs/core/src/context/world_state/mod.rs:301-354`): snapshot "should contain only the comparison data needed to decide what the model must be told next, and must not serialize to null because merge-patch nulls represent deletion"; `PreviousSectionState::{Absent, Unknown, Known}` decides re-send; legacy fragments recognized for migrated rollouts. Base instructions are emitted only when previous is `Absent` (`codex-rs/core/src/context/world_state/base_instructions.rs:12-27`).
- **When**: before EVERY sampling request (not only per user turn) the loop renders the step's world state and records only the diff vs the baseline as conversation items, then persists a `RolloutItem::WorldState` patch (`codex-rs/core/src/session/turn.rs:504-505`; `codex-rs/core/src/session/mod.rs:3614-3645`). Mid-turn changes (environment, AGENTS.md edits, model switch) reach the model as appended deltas.
- **Change wording**: AGENTS.md "These AGENTS.md instructions replace all previously provided AGENTS.md instructions." / "The previously provided AGENTS.md instructions no longer apply." (`codex-rs/core/src/context/world_state/agents_md.rs:11-13`, `:44-80`); context-window guidance analogous (`codex-rs/core/src/context/world_state/context_window_guidance.rs:8-10`); `<model_switch>` inserted first in the developer bundle (`codex-rs/core/src/session/mod.rs:4370-4375`).
- **Packing**: adjacent mergeable fragments of the same role become one message; `Placement::{Mergeable, Standalone, Prefix}` (`codex-rs/core/src/context_manager/updates.rs:1-62`).
- **Persistence**: `WorldStateItem{full, state}` — full snapshot = baseline, later entries RFC 7386 merge patches (`codex-rs/protocol/src/protocol.rs:3330-3347`; `codex-rs/core/src/context/world_state/mod.rs:412-456`). Turn-level baseline also in `TurnContextItem` / `reference_context_item`; missing baseline → full re-injection (`codex-rs/core/src/session/mod.rs:4612-4630`); rollback trimming a mixed initial-context developer message clears it (`codex-rs/core/src/context_manager/history.rs:983-995`).
- **After compaction**: `build_world_state_for_step(..., new_window=true)` renders everything in full, inserted before the last real user message (`codex-rs/core/src/session/turn.rs:1448-1451`; `codex-rs/core/src/compact.rs:80-111`).
- **Append-only deltas**: newly approved command prefixes / network rules as small developer notes ("Approved command prefix saved:", `codex-rs/core/src/context/approved_command_prefix_saved.rs:5`; `codex-rs/core/src/context/network_rule_saved.rs`) instead of re-sending permissions (`codex-rs/core/src/context/world_state/compact_permissions.rs:12`); incremental tool catalog notices (Responses Lite, `codex-rs/core/src/context/world_state/top_level_tools.rs:21-23`) → [[transcript-carried-system-prompt]].

## Constants
| name | value | path:line |
|---|---|---|
| `MAX_ENVIRONMENT_SUBAGENTS` | 8 (in `<subagents>` section) | `codex-rs/core/src/session/world_state.rs:40` |
| `MAX_ENVIRONMENT_SUBAGENT_BYTES` | 1_024 | `codex-rs/core/src/session/world_state.rs:41` |

## Evolution
- 2025-08-06 `3e8bcf0247` `<environment_context>` user message (static per turn).
- 2026-01-12 `87f7226cca` permissions as a developer message "injected at session start and any time sandbox/approval settings change".
- 2026-02-20 `bb0ac5be70` "Fix compaction context reinjection and model baselines (#12252)" — context updates recorded before pre-turn compaction got summarized away and not reinjected.
- 2026-06-22 `3b32d861c5` "[codex] migrate environment context to model world state (#29249)" — "initial injection, turn-to-turn updates, and changes that happen within a turn use different baselines".
- 2026-06-24 `3e51b46eba` / `fa036d39aa` serializable snapshots, world state persisted in rollouts (#29833, #29835); `a74771340d` "[3/3] core: replay persisted world state (#29837)"; `f2f80ef442` AGENTS.md replacement/removal notices.
- 2026-08-03 `1bbfb5cfad` "Avoid reinjecting permissions after command approvals" → [[permission-context-reinjected-repeatedly]].
- 2026-08-06 `a17da5e6e4` strip ModelSwitchInstructions/PersistentModeState from first-turn developer content on rollback → [[fork-carries-startup-context]].
- 2026-09-10 `935ac7710d` global AGENTS.md re-read at every model-request boundary → [[stale-context-files-mid-session]].
- 2026-10-03 `6326163b9a` incremental tool catalog (Responses Lite); 2026-10-05 `402f5b6fdf` / `93f8e79fd2` explicit update/removal semantics → [[tool-loadout-stale-within-run]].

## Versus pi
- pi re-derives request context from the session log each request ([[context-projection]]) and appends system-prompt *section patches* ([[transcript-carried-system-prompt]]); codex keeps a mutable live history and diffs typed snapshots per request, persisting merge patches so diffing survives resume/fork.
