---
type: implementation
harness: pi
concept: virtual-model-router
commit: b30a6dd77
files: [packages/coding-agent/src/core/virtual-models.ts:29, packages/coding-agent/src/core/model-runtime.ts:996, packages/coding-agent/src/core/agent-session.ts:812, packages/coding-agent/docs/virtual-models.md:84, packages/coding-agent/src/core/sdk.ts:367]
---
[[virtual-model-router]] in [[pi]].

## Mechanism
- Catalog entries with `api: "pi-virtual"` (`packages/coding-agent/src/core/virtual-models.ts:29`) that route every request to a physical model; "A virtual model never reaches a provider" (`:1-11`). Unrouted stream throws `must be routed before streaming` (`:189-194`).
- Registration: extension `registerVirtualModel` (`packages/coding-agent/src/core/extensions/types.ts:1881`; `540e174c7` #10035, experimental, `packages/coding-agent/CHANGELOG.md:251`). Definition: `provider, id, name, thinkingLevels (default ["off"]), contextWindow/maxTokens (0 = unknown), input (default text+image), route(request)` (`virtual-models.ts:84-102`). Thinking-level map marks unoffered levels `null` (`:170-187`) → same clamp machinery as physical models ([[pi--thinking-level-abstraction|thinking-level-abstraction]]).
- **Route reasons** `user | continuation | retry | direct` (`:43-50`). Request carries `previous` (latest successful physical response), `failed` (for retry), `state`, `messages`, `signal` (`:52-70`).
- `resolveModel` (`packages/coding-agent/src/core/model-runtime.ts:996-1027`): target must be physical and have credentials, else throws; thinking clamped via `clampThinkingLevel`.
- **Loop routing** (`packages/coding-agent/src/core/agent-session.ts:812-842`): reason = `retry` if a failed response exists, `user` if a user message follows the last assistant, else `continuation`; router state persisted as custom session entry `pi.virtual-model-state` when changed (`:829-835`; `virtual-models.ts:31-39, 158-167`) — stored before the request even if it fails, follows session tree, survives compaction (`docs/virtual-models.md:105-108`). If routed model's window is exceeded → threshold compaction then re-prepare; route stands (`agent-session.ts:838-841`). Failed message passed as `failed` from `_prepareRetry` (`:823-826`).
- Summaries/compaction route with reason `direct` (`agent-session.ts:582-589`). Direct `streamSimple` on a virtual model: route `direct`, cap `maxTokens` to routed model, **drop caller apiKey/headers/env when routed provider differs** "so they are not sent to the wrong vendor" (`model-runtime.ts:716-736`).
- Physical id collision: registration throws if a physical model already has that id (`model-runtime.ts:960-963`); later catalog refresh adding the same physical id is hidden by the virtual one (`virtual-models.ts:196-218`). Virtual-only provider shows as configured with `source:"virtual"` (`model-runtime.ts:967-973`).
- Docs guidance: return `previous` for continuation and `failed` for retry to keep prompt caches and thinking signatures valid (`docs/virtual-models.md:84`) — i.e. routing must respect [[signed-reasoning-replay]] / [[cache-stable-prompt-prefix]].
- Context accounting uses the physical model of the latest response (`docs/virtual-models.md:29`). Branch model selection: last `model_change` wins only if virtual, else latest physical response (`virtual-models.ts:120-145`).
- Cache warming skipped for routed/redirected requests: warm only when request model == selected model (`sdk.ts:367-381`) — see [[cache-warming]].
- Example router: `examples/extensions/jev-router.ts` — plans on Codex Sol/Terra via Jev classifier ([[structured-classifier-api]]), switches to Luna after first edit (`examples/extensions/README.md:132`). Core ships mechanism only; no client-side cross-provider failover (08-absences; only agent retry + server-side Anthropic fallback [[server-side-refusal-fallback]]).

## Constants
| name | value | path:line |
|---|---|---|
| virtual api id | `pi-virtual` | packages/coding-agent/src/core/virtual-models.ts:29 |
| router state entry | `pi.virtual-model-state` | packages/coding-agent/src/core/virtual-models.ts:31-39 |
| default thinkingLevels | ["off"] | packages/coding-agent/src/core/virtual-models.ts:84-102 |
| contextWindow/maxTokens unknown | 0 | packages/coding-agent/src/core/virtual-models.ts:84-102 |

## Evolution
- 2026-09-28 `540e174c7` virtual models (#10035), experimental.
- 2026-09-30 `a0660b174` branch model selection with one catalog lookup (prompt submission slowed with session length, #10198).

## Evidence commits
540e174c7, a0660b174

## Quirks
- Physical model of the latest response, not the virtual id, drives context accounting and overflow checks — virtual windows of 0 mean "unknown".
- Router state committed before the request → a failing request still advances router state (by design, docs `:105-108`).

## Failures
- [[catalog-hot-path-quadratic]]
