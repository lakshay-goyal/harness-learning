---
type: implementation
harness: pi
concept: deferred-responses
commit: b30a6dd77
files: [packages/ai/src/types.ts:247, packages/ai/src/types.ts:506, packages/ai/src/models.ts:918, packages/ai/src/api/lazy.ts:69, packages/ai/src/utils/event-stream.ts:91, packages/ai/src/providers/faux.ts:543, packages/durable/src/harness/generation.ts:73]
---
[[deferred-responses]] in [[pi]].

## Mechanism
- Request: `SimpleStreamOptions.deferred: boolean | {window: "15m"|"1h"|"24h"}` asks a capable provider to return a durable handle and stop with `stopReason:"deferred"` (`packages/ai/src/types.ts:360-361, 456`).
- `DeferredHandle{provider, modelId, api, id, expiresAt?, pollAfterMs?, data?}` (`types.ts:506-516`) carried on `AssistantMessage.deferred` (`types.ts:552-583`); `done` event reason may be `deferred` (`types.ts:788-804`).
- Retrieval: `Models.streamDeferred(handle)` / `fetchDeferred(handle, {wait})` — `wait` = long-poll ms, default 0 = single status check — and best-effort `cancelDeferred` (`types.ts:247-256`; `packages/ai/src/models.ts:918-952`). API modules optionally export `fetchDeferred/cancelDeferred` in the `ProviderStreams` contract (`types.ts:292-305`); lazy wrappers declare capability flags (`packages/ai/src/api/lazy.ts:69-97`).
- **No production adapter implements it at HEAD**: `fetchDeferred`/`cancelDeferred` appear only in `packages/ai/src/types.ts`, `models.ts`, `api/lazy.ts` (capability plumbing, no caller passes `fetchDeferred: true`) and the test double `providers/faux.ts` (grep `fetchDeferred` over `packages/ai/src`); `api/anthropic-messages.ts` mentions "deferred" only for deferred *tools* (`__pi_deferred_placeholder__`, `anthropic-messages.ts:198-205`), not deferred responses. Not implemented by OpenAI-family adapters either, despite the original draft being OpenAI background mode (`382aa641c` "DRAFT: add openai background mode responses", #7339).
- Timing: `AssistantMessageEventStream` stamps `durationMs` with a monotonic clock unless message already timed or `timestamp < stream start` — so deferred results fetched later stay untimed (`packages/ai/src/utils/event-stream.ts:91-130`).
- Test double: faux provider supports deferred (`fetchDeferred`/`cancelDeferred`, `pendingFetches`, `pollAfterMs`) (`packages/ai/src/providers/faux.ts:543-661`).
- Classic coding-agent loop: `deferred` stop treated as non-error/non-tool → natural stop (inferred; retrieval path not traced, unverified).

## Durable variant (packages/durable)
- First-class phase: `pi.generation` checkpoint phases `prepare`, `request`, `retry`, **`poll`**, `tools` (`packages/durable/src/harness/generation.ts:53-83`). A `deferred` stop goes to `poll` with `pollAfterMs ?? 5000` (`generation.ts:109, 432`); poll uses `models.fetchDeferred/cancelDeferred` (`spec.md:3308-3310, 3566-3574`).
- Handle persisted in checkpoint state → survives process restart; context derivation excludes assistant messages with `deferred` stop reason from requests (`harness/context.ts:9`).

## Constants
| name | value | path:line |
|---|---|---|
| deferred windows | 15m / 1h / 24h | packages/ai/src/types.ts:360-361 |
| `fetchDeferred` default wait | 0 (single status check) | packages/ai/src/types.ts:247-256 |
| durable default poll delay | 5000 ms (`pollAfterMs ?? 5000`) | packages/durable/src/harness/generation.ts:109,432 |

## Evolution
- 2026-08-04 `382aa641c` deferred (background) responses (draft started as OpenAI background mode, #7339); same day `fed6009cc` caller-owned cancellation across auth/catalog refresh.
- Durable Pico5 harness adds `poll` phase (spec; `generation.ts`).

## Evidence commits
382aa641c, fed6009cc

## Quirks
- Only Anthropic implements fetch/cancel; OpenAI background mode (origin of the feature) not wired at HEAD (unverified why).
- Coding-agent's handling of a `deferred` stop (resume UI, polling) not found (unverified).

## Failures
- (none mined)
