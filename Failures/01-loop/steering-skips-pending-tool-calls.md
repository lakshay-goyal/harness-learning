---
type: failure
concepts: [steering-queue, parallel-tool-execution]
harnesses: [pi]
---
**Symptom** — A user message typed mid-run caused the remaining tool calls of the current assistant message to be skipped and answered with synthetic error results "Skipped due to queued user message." — the model issued N calls and got fewer real results.

**Root cause** — "Queued message steering" (`117af076c` 2025-12-20, `packages/ai/CHANGELOG.md:2155`, 0.25.1): `executeToolCallsSequential` polled `getSteeringMessages()` after every tool and, on a hit, called `skipToolCall` for the rest of the batch.

**Fix · [[pi]]** — `208a2cc12` 2026-03-16 "defer steering until after tool execution": per-tool poll removed; steering read once after the whole batch (`packages/agent/src/agent-loop.ts:295` at HEAD); contract "Tool calls from the current assistant message are not skipped" (`packages/agent/src/types.ts:285-293`); CHANGELOG `packages/agent/CHANGELOG.md:497`. Docs `8a8e2a804` 2026-03-18 (#2330).

**Lesson** — Deliver soft interrupts at turn boundaries; never leave a model-issued tool batch half-executed — only a hard abort may cut tools.

Related: [[steering-queue]] · [[parallel-tool-execution]] · [[abort-propagation]] · [[pi--steering-queue|pi]]
