---
type: implementation
harness: opencode
concept: steering-queue
commit: ecc4916b5a
files: [packages/opencode/src/session/prompt.ts:1052-1071, packages/opencode/src/session/prompt.ts:1106-1115, packages/opencode/src/effect/runner.ts:115-138, packages/core/src/session.ts:359-385, packages/core/src/session/input.ts:41-288, packages/core/src/session/runner/llm.ts:186-196]
---
[[steering-queue]] in [[opencode]].

## Mechanism

### Legacy runtime — steering falls out of the store-driven loop
- `SessionPrompt.prompt` persists the user message immediately (`createUserMessage`), then calls `loop` (`packages/opencode/src/session/prompt.ts:1052-1071`). If a run is in flight, `Runner.ensureRunning` on a `Running` runner just awaits that run's `Deferred` — no second loop (`packages/opencode/src/effect/runner.ts:115-138`).
- The running loop re-reads history at the top of the next iteration; the new message becomes `lastUser`, the exit test fails on `lastAssistant.parentID !== lastUser.id` (`prompt.ts:1115`), so the model is called again with the new message at the end.
- Delivery point = after the current step's whole tool batch (tools run inside `streamText`); in-flight tools are never cancelled.
- No separate queue object, no one-at-a-time vs all mode: every persisted message is visible at the next step.
- `step` is per run, so steered messages share the running step counter (legacy).

### v2 runtime — durable admission inbox
- `sessions.prompt` admits one event-sourced `session_input` row, then `wake`s the coordinator unless `resume:false`; default `delivery` is `"steer"` (`packages/core/src/session.ts:359-385`, default at `:366`).
- Idempotent by message id: exact retry returns the same receipt; reuse with different prompt/delivery → `PromptConflictError` (`packages/core/src/session/input.ts:41-81`; `specs/v2/session.md:13-20`).
- **Admitted Prompt vs Prompt Promotion**: steers promote at the next safe provider-turn boundary only if admitted at or before the cutoff sequence (`input.ts:245-266`; cutoff `packages/core/src/session/runner/llm.ts:188`). Promotion is one durable `Prompted` event whose projector writes the user message and marks the row promoted in one transaction (`specs/v2/session.md:35`).
- A pending steer alone keeps the inner loop running (`llm.ts:410`). Promoting ≥1 input resets the step allowance to 1 (`llm.ts:195`, `dc468bdcfd`).
- Rationale: "Prompt admission and model-visible promotion must be separate durable operations" (`specs/v2/schema-changelog.md:127-128`).

## Constants
| name | value | path:line |
|---|---|---|
| v2 default delivery | `"steer"` | `packages/core/src/session.ts:366` |

## Evolution
- 2026-01-02 `f991fbbde8` queued messages (step > 1) wrapped ephemerally in `<system-reminder>The user sent the following message: … Please address this message and continue with your tasks.</system-reminder>`, only in that request.
- 2026-06-19 `f092bafe88` "remove steering wrapper that can bust cache" — the per-request rewrite changed already-sent user text, so the next prefix missed the cache.
- 2026-06-04 `76ecf2e58c` v2 inputs event-sourced; 2026-06-22 `f48f24ec4e` promotion simplified.
- 2026-06-23 `dc468bdcfd` steps reset for promoted prompts → [[step-budget-not-reset-on-new-input]].

## Quirks / drift
- Legacy steering is structural (single fiber per session + re-read) rather than a queue, so the prevention of [[reentrant-prompt-corrupts-state]] needs no lock; no dedicated fix commit found.
- The wrapper removal is a caching lesson: transforms of history must be stable for a message's lifetime (owned by 06-caching).

Contrast: [[pi--steering-queue|pi]] holds an explicit `PendingMessageQueue` (one-at-a-time default) polled after each tool batch; opencode legacy needs no queue because the next step re-reads SQLite, and v2 makes the inbox durable with a cutoff sequence.
