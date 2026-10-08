---
type: implementation
harness: pi
concept: tool-argument-repair
commit: b30a6dd77
files: [packages/agent/src/agent-loop.ts:694, packages/agent/src/types.ts:470, packages/coding-agent/src/core/tools/edit.ts:103, packages/coding-agent/src/core/tools/tool-definition-wrapper.ts:8, packages/ai/src/utils/validation.ts:317, packages/durable/src/tools/edit.ts:45]
---
[[tool-argument-repair]] in [[pi]].

## Mechanism
Two layers before execution, in this order (`prepareToolCall`, `packages/agent/src/agent-loop.ts:708-777`): (1) per-tool `prepareArguments` shim, (2) generic schema-driven coercion + validation in pi-ai; then `beforeToolCall` ([[tool-call-gate]]).
- **Layer 1 — `prepareArguments`** (`AgentTool.prepareArguments?: (args: unknown) => Static<TParameters>`, "Optional compatibility shim for raw tool-call arguments before schema validation", `packages/agent/src/types.ts:470-473`). Called by `prepareToolCallArguments` (`agent-loop.ts:694-707`); identity return = no copy. Passed through `wrapToolDefinition` (`packages/coding-agent/src/core/tools/tool-definition-wrapper.ts:8-30`) so extensions get it too (`registerTool`, 09-platform §2.3). Introduced `b5f425ad1` (2026-03-29) "to accept legacy arg shapes from resumed sessions without polluting the public schema".
- Only built-in user: edit's `prepareEditArguments` (`packages/coding-agent/src/core/tools/edit.ts:103-134`):
  - `edits` is a JSON **string** → `JSON.parse`; array kept, single edit object wrapped (comment: "Some models (Opus 4.6, GLM-5.1) send edits as a JSON string instead of an array.") (`:110-118`; `a2ec01e12` #3370 — models otherwise "fall back to sed/python").
  - `edits` is a single `{oldText,newText}` object → `[obj]` (`:119-121`; `ca21c1686` #7835/#8011).
  - Legacy top-level `oldText`/`newText` → appended to `edits` and stripped (`:124-133`; `b5f425ad1`, after single-mode schema removal `e773527b3` #2639).
  - Parse failure silently ignored → validator reports the real error.
- **Layer 2 — `validateToolArguments`** (`packages/ai/src/utils/validation.ts:317-349`): `structuredClone` args → `normalizeOptionalNulls` deletes `null` on optional non-nullable props (`:239-269`; arrived with strict-schema conversion `7915cdac6`, where strict mode makes optionals nullable) → TypeBox `Value.Convert` → for non-TypeBox (plain JSON Schema, e.g. MCP) `coerceWithJsonSchema` (`:194-236`): primitive coercion by type (`coercePrimitiveByType` `:59-131`, numeric strings → numbers `923b9cb9e` #786), recursive object/array, unions try **values already matching an arm first** then coerced candidates (`coerceWithUnionSchema` `:175-192`; `2e95584da` #7373/#7328 — nullable union turned `null` into another primitive).
  - Failure throws `Validation failed for tool "X":\n  - path: msg\n\nReceived arguments:\n{json}` (`:340-348`) → becomes error tool result ([[tool-error-as-result]]).
- **Schema leniency**: edit item objects use `{}` options = `additionalProperties` allowed (`a1b336d73` #6278: model-invented extra fields rejected valid edits). Omitted inputs default `{}` for Anthropic/Google history (`af813f904` #1065).
- **Range validation instead of clamping**: bash `timeout` must be finite >0 and ≤ `MAX_TIMEOUT_MS` else clear error (`85b7c2474`, `cbcf4e04c` #6181) — see [[pi--shell-execution]].
- Display-only repair: read with `null` offset/limit rendered `:1` → fixed in renderer (`49681e1b7` #9996).

## Constants
| name | value | path:line |
|---|---|---|
| bash `MAX_TIMEOUT_MS` | 2_147_483_647 ms | `packages/coding-agent/src/core/tools/bash.ts:22` |

## Evolution
- 2026-01-16 `923b9cb9e` (#786) coerce string numbers in validation.
- 2026-01-30 `af813f904` (#1065) default tool-call arguments when providers omit inputs.
- 2026-03-27/28 `20a57e759` → `e773527b3` edit dual-mode added then collapsed to `edits[]` (#2639).
- 2026-03-29 `b5f425ad1` `prepareArguments` hook (agent + coding-agent); legacy edit shape folded.
- 2026-03-19 `0d7c81ec9` (#2395) skip AJV validation in restricted runtimes (CSP/eval, Cloudflare Workers).
- 2026-04-18 `a2ec01e12` (#3370) stringified `edits`.
- 2026-04-22 `35ff2689e` (#3474) TypeBox v1 migration; JSON-schema coercion for extension (non-TypeBox) schemas.
- 2026-07-04 `a1b336d73` (#6278) allow extra replacement fields.
- 2026-08-03 `2e95584da` (#7373, CHANGELOG #7328) union arms preserved before coercion.
- 2026-08-11 `7915cdac6` strict schema conversion + `normalizeOptionalNulls`.
- 2026-08-17 `ca21c1686` (#8011, #7835) single edit object.

## Evidence commits
`923b9cb9e`, `0d7c81ec9`, `af813f904`, `20a57e759`, `e773527b3`, `b5f425ad1`, `a2ec01e12`, `35ff2689e`, `a1b336d73`, `2e95584da`, `7915cdac6`, `ca21c1686`, `85b7c2474`, `cbcf4e04c`, `49681e1b7`

## Quirks
- Coding-agent `prepareEditArguments` **mutates the incoming args object in place** (`edit.ts:108,115-122`); durable copy explicitly works on a copy ("Works on a copy; the call's arguments stay unchanged", `packages/durable/src/tools/edit.ts:45-51`). Persisted tool-call args in the transcript therefore may already show the repaired shape (inferred, unverified).
- Validator errors echo the full received args back to the model (useful for self-correction, costs tokens on large edits).
- No repair layer for other built-ins (read/bash/write/grep/find/ls) beyond generic coercion.
- Constrained sampling ([[constrained-tool-sampling]], strict "prefer" default for read/bash/powershell/edit/write since `fcff255b0`) reduces shape errors upstream; repair is the fallback for providers without strict mode.

## Durable variant (packages/durable)
- Same three repairs (`edits` string / single object / top-level `oldText,newText`) in `prepareArguments`, on a copy (`packages/durable/src/tools/edit.ts:45-71`); harness `call` phase re-validates after `beforeTool` rewrites (`packages/durable/src/harness/tool.ts:55-90`).

## Failures
- [[tool-arg-shape-drift]]
- [[tool-arg-coercion-breaks-unions]]
- [[edit-tool-dual-mode-confusion]]
- [[bash-timeout-clamped-to-immediate]]
- related (02): [[strict-tool-schema-rejections]]
