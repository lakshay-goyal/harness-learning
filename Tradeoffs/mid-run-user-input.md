---
type: tradeoff
concepts: [steering-queue, follow-up-queue, out-of-band-message-deferral, event-sourced-session-store, cache-stable-prompt-prefix]
harnesses: [pi, opencode]
---
# mid-run-user-input

**Axis**: when the user types while the agent is working, where does the message go and when does the model see it?

| option | pi | opencode | evidence |
|---|---|---|---|
| In-memory steering queue, injected after the current tool batch | ✅ `steer()`; polled at run start and after each tool batch; default `one-at-a-time` | — | pi: `packages/agent/src/agent.ts:298-301`; `packages/agent/src/agent-loop.ts:295` → [[steering-queue]] |
| Separate follow-up queue (only when the agent would stop) | ✅ `followUp()` | implicit (same mechanism) | pi `packages/coding-agent/src/core/settings-manager.ts:864` → [[follow-up-queue]] |
| Persist the user message at once; loop picks it up at next step | — | ✅ legacy: `prompt` writes the user message, `ensureRunning` joins the in-flight run; next iteration sees a newer `lastUser` and continues | `packages/opencode/src/session/prompt.ts:1052-1071`; `packages/opencode/src/effect/runner.ts:115-138` |
| Durable inbox with explicit delivery | experimental durable inbox (`packages/durable/src/harness/inbox.ts:16`) | ✅ v2: admitted rows, `delivery: "steer"` (default, next safe turn boundary) or `"queue"` (FIFO when idle); promotion is one durable event | `packages/core/src/session.ts:359-366`; `specs/v2/session.md:155-158`; `76ecf2e58c` |
| Wrapper text around queued messages | — | tried `<system-reminder>The user sent the following message…` (`f991fbbde8` 2026-01-02), removed because it busted the cache (`f092bafe88` 2026-06-19) | → [[ephemeral-history-rewrite-busts-cache]] |
| Abort behavior | Esc returns queued text to the editor, then aborts | queued message is already persisted; it stays | pi `docs/how-pi-works.md:13` |

**When each wins**
- **Queue in memory (pi)**: precise control (steer vs follow-up, one-at-a-time), cancel-and-edit on Esc. Lost if the process dies.
- **Persist first (opencode legacy)**: zero extra state, crash-safe, and any client (TUI, web, ACP) can post. Cost: no steer/follow-up distinction; queued messages share the run's step counter.
- **Durable inbox (opencode v2)**: multi-client servers and idempotent retries (same message id = same receipt). Cost: an event model to design. Lesson from both runtimes: never rewrite already-sent history to "frame" a queued message; append it as-is.

Related: [[steering-queue]] · [[client-server-vs-single-process]] · [[prompt-cache-strategy]] · [[pi]] · [[opencode]] · [[Tradeoffs]]
