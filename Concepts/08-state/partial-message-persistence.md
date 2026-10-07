---
type: concept
stage: state
tier: candidate
aliases: ["stopReason: aborted", "stopReason: error", "stopReason: pending", AssistantMessageFrameEncoder, AssistantMessageFrame, partialIntervalMs, persistable-stream-frames, turn_aborted, OutputItemDone, generic.turn_aborted]
harnesses: [pi, codex]
---
Policy for interrupted or failed model output: what of a half-streamed assistant message is persisted (and when), and how it is excluded or converted when the history is replayed to a provider.

## Why
- Discarding interrupted output loses what the user saw (and paid for); replaying it verbatim breaks providers (empty 429/500 turns break tool_use→tool_result chains `fbb74bb29`; unsigned partial thinking rejected `387cc97ba`; scratch `partialJson` corrupts resumes `e2b40dfc8`).
- Crash-resumable runtimes must commit partials periodically or lose everything since the last commit; commit cadence must not change semantics ([[output-window-depends-on-commit-cadence]]).
- Session switches mid-turn must settle (abort + persist) first or leave dangling tool calls (`cefa40ed8`).

## Design space
- **Persist nothing until done** vs **persist the final partial with `stopReason: aborted|error`** (pi coding-agent) vs **commit streaming partials every N ms** (pi-durable, 100 ms).
- **Never persist `pending` partials** (pi) vs persist.
- **Replay policy**: skip errored/aborted assistant turns at the provider boundary + synthesize orphan tool results (pi `transformMessages`) vs exclude in projection (durable) vs replay with downgraded content (unsigned thinking → text).
- **Scratch-field stripping**: remove stream-only buffers (`partialJson`, `index`, `redactedChunks`) on every terminal path before persistence.
- **Persistable frames**: compact event frames that skip already-covered deltas for SDK consumers (pi-ai `AssistantMessageFrameEncoder`).
- **Recovery of leftover committed partial**: convert to aborted entry before re-requesting (durable `request` phase).
- **Persist completed items only, never partial deltas** ✔ codex (items recorded on `OutputItemDone`; half items never exist in history).
- **Explicit abort marker in history** (model-visible `<turn_aborted>` user-role fragment) ✔ codex — see [[interrupted-turn-invisible-to-model]].
- **Synthetic outputs for aborted/unanswered tool calls** ✔ codex.

## Implementations
- [[pi--partial-message-persistence|pi]] — aborted/error partial persisted on `message_end`, skipped by `transformMessages`; `pending` never written; durable commits partials every 100 ms and converts leftovers to aborted on recovery.
- [[codex--partial-message-persistence|codex]] — never persists half messages: completed output items recorded as they arrive, aborted tools get synthetic outputs, interrupt leaves a model-visible `<turn_aborted>` marker flushed before `TurnAborted`.

## Failures
- [[output-window-depends-on-commit-cadence]]
- [[turn-items-lost-on-abort]]
- [[interrupted-turn-invisible-to-model]]
- [[session-lost-before-first-response]] (08-state) — Exiting (Ctrl+C, crash) during the first turn of a new session lost the session entirely, including the…

## Related
[[errors-as-stream-events]] · [[transcript-replay-repair]] · [[signed-reasoning-replay]] · [[abort-propagation]] · [[session-tree]] · [[durable-execution]] · [[context-projection]] · [[truncated-tool-call-guard]]
