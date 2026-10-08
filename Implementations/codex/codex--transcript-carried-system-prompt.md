---
type: implementation
harness: codex
concept: transcript-carried-system-prompt
commit: 622e9e3696
files: [codex-rs/core/src/client.rs:913-941, codex-rs/core/src/client.rs:884-887, codex-rs/core/src/client.rs:998, codex-rs/core/src/client.rs:176-177, codex-rs/core/src/context/world_state/base_instructions.rs:1-27, codex-rs/core/src/context/world_state/top_level_tools.rs:21-23, codex-rs/core/src/context/world_state/agents_md.rs:11-13, codex-rs/core/src/session/mod.rs:4370-4375, codex-rs/core/src/session/reasoning_effort.rs:28-91]
---
[[transcript-carried-system-prompt]] in [[codex]]. This is a partial match. The general mechanism, typed world-state sections diffed per sampling step, is [[world-state-diff-injection]]. This note covers how the *system prompt and tool declarations themselves* are carried as transcript items.

## Mechanism
- **Base instructions are an input item, not the `instructions` field.**
  - Each request splices a `developer` message `BaseInstructionsFragment` at the head of `input`, with a stable UUIDv5 id (`codex-rs/core/src/client.rs:930-941`; `c9253c4977` 2026-10-05).
  - The world-state section `base_instructions` "records fixed base instructions alongside tool declarations at the start of each window". It emits its fragment only when the previous state is `Absent`, i.e. at the start of a window, and never as a mid-window patch (`codex-rs/core/src/context/world_state/base_instructions.rs:1-27`).
  - How these two paths interact (whether one supersedes the other per feature flag) is unverified.
- **Tool declarations as a transcript item (Responses Lite).**
  - When `model_info.use_responses_lite` is set:
    - tools are sent as an `AdditionalTools` developer input item (id UUIDv5 of the tools JSON) instead of the top-level `tools` field;
    - `parallel_tool_calls` is forced false;
    - `reasoning.context = all_turns` is set;
    - the transport header `x-openai-internal-codex-responses-lite` is added.

    Code: `codex-rs/core/src/client.rs:913-929,884-887,998,176-177`.
  - Lite has no hosted tools, so web search and image generation run through Codex-owned standalone executors (`ffe90cb5c3` 2026-06-05). Image detail is stripped for Lite (`4435ff2810` 2026-06-10).
  - Introduced in `954e2878bb` 2026-06-05.
- **Incremental tool catalog** (`IncrementalTools`, Responses Lite; `6326163b9a` 2026-10-03, #50540). The initial catalog is sent once. Later only added or changed definitions are appended to history, with explicit merge-semantics text (`codex-rs/core/src/context/world_state/top_level_tools.rs:21-23`):
  - "This is an incremental namespace update. Previously declared tools remain available for direct calls unless explicitly marked unavailable. If a tool is redefined here, its latest definition replaces the earlier one."
  - "The following tools are no longer available. Do not call them:"
  - "The following namespaces are no longer available. Do not call tools in them unless those tools are declared in a later update:"
- **Replacement notices for instruction files.** "These AGENTS.md instructions replace all previously provided AGENTS.md instructions." (`codex-rs/core/src/context/world_state/agents_md.rs:11-13`; `935ac7710d` 2026-09-10).
- **Model switch.** A `<model_switch>` fragment is inserted first in the developer bundle (`codex-rs/core/src/session/mod.rs:4370-4375`).
- **Sampling parameters are carried as items too.** Effort changes become `ResponseItem::ConfigurationUpdate` with `harness_authored_configuration: true` (`codex-rs/core/src/session/reasoning_effort.rs:28-91`). See [[cache-preserving-config-update]].

## Evolution
- `3b32d861c5` 2026-06-22: environment context migrated to model world state. `a74771340d` 2026-06-24: persisted world state is replayed.
- `954e2878bb` / `ffe90cb5c3` 2026-06-05: Responses Lite transport and standalone hosted-tool executors. `4435ff2810` 2026-06-10: image detail stripped for Lite.
- `56a8470aa0` 2026-09-05: configuration-update items.
- `935ac7710d` 2026-09-10: AGENTS.md reloaded at each request boundary as appended replacement notices.
- `6326163b9a` 2026-10-03: incremental tool catalog.
- `402f5b6fdf` 2026-10-05: hints stating that earlier tools remain available.
- `93f8e79fd2` 2026-10-05: namespace removals split from per-tool removals ("Removing a whole namespace previously listed both the namespace and every member as unavailable tools").
- `c9253c4977` 2026-10-05: base instructions moved into `input`.

## Versus pi
- [[pi--transcript-carried-system-prompt|pi]] stores a `SystemMessage{sections, toolsAdded, toolsRemoved}` in the session log and appends section patches.
- Codex has no section-patch format for the base prompt. The base prompt is regenerated per request with a deterministic id, and is fixed per window. Everything that *changes* is a separate typed world-state section with replace/remove notices.
- Native tool deltas exist only on Responses Lite (`AdditionalTools` plus incremental updates). On standard Responses the full `tools` array is resent.

## Failures
- [[tool-loadout-stale-within-run]]
- [[stale-context-files-mid-session]]
- [[permission-context-reinjected-repeatedly]]
