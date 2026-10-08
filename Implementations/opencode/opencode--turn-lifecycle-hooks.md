---
type: implementation
harness: opencode
concept: turn-lifecycle-hooks
commit: ecc4916b5a
files: [packages/opencode/src/session/prompt.ts:1255, packages/opencode/src/session/llm/request.ts:114-140, packages/opencode/src/session/tools.ts:106-128, packages/opencode/src/session/compaction.ts:374-379]
---
[[turn-lifecycle-hooks]] in [[opencode]].

## Mechanism (legacy runtime; plugin hooks)
- `experimental.chat.messages.transform` — fired before every model step with the mutable message list (`packages/opencode/src/session/prompt.ts:1255`) and before compaction (`packages/opencode/src/session/compaction.ts:379`). Rewrites the next request.
- `chat.params` — plugins mutate `temperature`, `topP`, `topK`, `maxOutputTokens`, provider `options` per request (`packages/opencode/src/session/llm/request.ts:114-130`).
- `chat.headers` — per-request headers (`request.ts:134-140`).
- `tool.execute.before` (can mutate args) / `tool.execute.after` (can mutate output) around every tool (`packages/opencode/src/session/tools.ts:106-128`).
- `experimental.session.compacting` — compaction prompt hook (`compaction.ts:374`).
- No hook can end or force-continue a step: decision vocabulary is "rewrite", never end/continue.

## Constants
none.

## Evolution
- Not traced (hook surface is part of the plugin API; see [[extension-event-hooks]]).

## Quirks / drift
- v2 `PluginV2.HookSpec` hooks are catalog/provider oriented (`provider.update`, `model.update`) per `specs/v2/catalog-config-plugin-lifecycle.md`; no per-turn hook found in `packages/core/src/session/runner` (unverified absence).

Failures: [[tool-loadout-stale-within-run]] (avoided: tools resolved per step).

Contrast: [[pi--turn-lifecycle-hooks|pi]] hooks can end or continue a turn (`finishTurn`); opencode hooks only rewrite messages, params, headers and tool I/O.
