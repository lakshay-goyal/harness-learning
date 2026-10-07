---
type: implementation
harness: pi
concept: token-estimation
commit: b30a6dd77
files: [packages/coding-agent/src/core/compaction/compaction.ts:140, packages/coding-agent/src/core/compaction/compaction.ts:196, packages/coding-agent/src/core/compaction/compaction.ts:226, packages/coding-agent/src/core/compaction/compaction.ts:276, packages/coding-agent/src/core/compaction/compaction.ts:298, packages/ai/src/utils/estimate.ts:15, packages/ai/src/utils/estimate.ts:71, packages/coding-agent/src/core/agent-session.ts:3048, packages/durable/src/harness/compaction.ts:307]
---
[[token-estimation]] in [[pi]].

## Mechanism
- **Per-message heuristic (coding-agent)** `estimateTokens` = `ceil(chars/4)`, comment "conservative (overestimates)" (`packages/coding-agent/src/core/compaction/compaction.ts:294-349`): system (content + sections + `JSON.stringify(toolsAdded)`), user, assistant (text + thinking + toolCall name + args JSON), custom/toolResult, bashExecution (command + output), branch/compaction summaries (summary text). Images: `ESTIMATED_IMAGE_CHARS = 4800` per image block (`packages/coding-agent/src/core/compaction/compaction.ts:276-292`).
- **Usage reading**: `calculateContextTokens(usage) = totalTokens || input+output+cacheRead+cacheWrite` (`packages/coding-agent/src/core/compaction/compaction.ts:140-142`); `getAssistantUsage` skips aborted, error and all-zero usage (`packages/coding-agent/src/core/compaction/compaction.ts:148-162`).
- **Hybrid** `estimateContextTokens(messages)`: last valid assistant usage + chars/4 of every later message; no usage → pure estimate, `lastUsageIndex=null` (`packages/coding-agent/src/core/compaction/compaction.ts:196-224`).
- **Boundary-aware** `estimateProjectedContextTokens(projection, branch)` (`packages/coding-agent/src/core/compaction/compaction.ts:226-262`): maps the usage-bearing projected message back to its source entry; trusts usage only if that entry is *after* the latest `context_edit` or `compaction` on the branch; otherwise pure estimate of current system message + all non-system messages. Comment: "without trusting usage captured before a later edit or compaction".
- **Session use** (`packages/coding-agent/src/core/agent-session.ts:3048-3083`): projection has edits → projected estimate; error/zero usage → `estimateContextTokens(agent.state.messages)` with guard "usage message timestamp ≤ latest compaction → return false"; else direct usage. Mid-run threshold uses the projected estimate (`packages/coding-agent/src/core/agent-session.ts:766-774`).
- `tokensBefore` on the compaction entry = projected estimate at preparation (`packages/coding-agent/src/core/compaction/compaction.ts:897`); `estimatedTokensAfter` = pure estimate of the new projection (`packages/coding-agent/src/core/agent-session.ts:2849`, `packages/coding-agent/src/core/agent-session.ts:3179`, type `packages/coding-agent/src/core/agent-session.ts:366-372`); footer shows `?/200k` until next real usage (`7eb969ddb`).
- **pi-ai estimator** (`EST`), used by the request max-tokens clamp ([[max-tokens-context-clamp]]): `CHARS_PER_TOKEN = 3.5`, image 4800 chars (~1371 tok) (`packages/ai/src/utils/estimate.ts:15-16`); system = text + `toolsAdded`/`toolsRemoved` JSON; `getLastAssistantUsageInfo` ignores usage from an assistant older than any later-inserted prefix message (timestamp scan; e.g. a compaction summary) (`packages/ai/src/utils/estimate.ts:71-95`); hybrid total (`packages/ai/src/utils/estimate.ts:97-112`).

## Constants
| name | value | path:line |
|---|---|---|
| coding-agent chars/token | 4 | `packages/coding-agent/src/core/compaction/compaction.ts:298-349` |
| pi-ai `CHARS_PER_TOKEN` | 3.5 (was 4 until `27075fe07`) | `packages/ai/src/utils/estimate.ts:15` |
| `ESTIMATED_IMAGE_CHARS` | 4800 (both estimators, duplicated) | `packages/coding-agent/src/core/compaction/compaction.ts:276`, `packages/ai/src/utils/estimate.ts:16` |
| durable estimator | pi-ai `estimateMessageTokens` (3.5) | `packages/durable/src/harness/compaction.ts:307-322` |
| transcript script cap | `MAX_CHARS_PER_FILE = 100_000` "~20k tokens" (implies 5 chars/tok) | `scripts/session-transcripts.ts:20` |

## Evolution
- 2026-02-12 `7eb969ddb` (#1382) latest (not first) compaction boundary; unknown usage display after compaction.
- 2026-03-05/06 `a4f4d91fa`, `d5c18e024` (#1860), `d1a17bbae` ignore stale pre-compaction usage (threshold path and error path).
- 2026-03-06 `b8910f13a` (#1834) error messages estimate from last successful usage (persistent 529s never compacted).
- 2026-05-26 `96f0edd02` (#4983) count user image tokens.
- 2026-07-09 `a6f720e6c` (#6326) count custom messages in compaction budget; `8973ae28a` (#6464) pi-ai ignores usage older than a later prefix insertion.
- 2026-08-19 `4495469a5` (#8328) providers without usage: pure size estimate (previously `return false`, never compacted).
- 2026-09-21 `466db0fec` projected estimate trusts usage only after latest edit/compaction.
- 2026-10-06 `27075fe07` (#10497) pi-ai 4 → 3.5 chars/token after DeepSeek V4 Flash overflow.

## Evidence commits
`7eb969ddb` `a4f4d91fa` `d5c18e024` `d1a17bbae` `b8910f13a` `96f0edd02` `a6f720e6c` `8973ae28a` `4495469a5` `466db0fec` `27075fe07` `c60f6a8ab`

## Quirks
- Two heuristics disagree: compaction's "conservative" chars/4 now *under*-estimates relative to pi-ai's 3.5 (07-constants observation) — threshold may fire later than the request clamp expects.
- `ESTIMATED_IMAGE_CHARS` duplicated in two packages (drift risk).
- Silent-overflow and usage include `cacheRead`; tool declarations count only when carried in system deltas.

## Durable variant (packages/durable)
- `estimateContext(view, extra)` (`packages/durable/src/harness/compaction.ts:307-322`): usage of the newest assistant appended *after the head marker* (whose request included the marker) + `estimateMessageTokens` of later messages + planned system entries; none → estimate everything.

## Failures
[[stale-usage-drives-compaction]] · [[compaction-starved-by-missing-usage]] · [[estimator-undercounts-context]]
