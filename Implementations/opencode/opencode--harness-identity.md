---
type: implementation
harness: opencode
concept: harness-identity
commit: ecc4916b5a
files: [packages/opencode/src/session/system.ts:76, packages/opencode/src/session/prompt/anthropic.txt:1, packages/opencode/src/session/prompt/gpt.txt:1, packages/opencode/src/session/prompt/meta.txt:1, packages/opencode/src/session/prompt/beast.txt:1, packages/core/src/plugin/agent.ts:12-13, origin/v2:packages/core/src/session/runner/prompt/system.txt:1, origin/v2:packages/core/src/plugin/identity.ts:9-30]
---
[[harness-identity]] in [[opencode]].

## Mechanism
### Legacy runtime
- Every family prompt opens with a **harness-as-self** identity: "You are OpenCode, the best coding agent on the planet." (`packages/opencode/src/session/prompt/anthropic.txt:1`, codex.txt:1), "You are OpenCode, You and the user share the same workspace…" (gpt.txt:1), "You are opencode, an agent - please keep going…" (beast.txt:1), "You are OpenCode … You are powered by {{MODEL_NAME}}, a large language model trained by Meta MSL" (meta.txt:1) → [[per-model-system-prompt]].
- Model identity stated separately as a fact, first line of the env block: "You are powered by the model named ${model.api.id}. The exact model ID is ${providerID}/${api.id}" (`packages/opencode/src/session/system.ts:76`) → [[xml-prompt-boundaries]].
### v2 runtime (dev)
- Built-in build agent system: "You are an AI coding agent. Help the user accomplish software engineering tasks…" — no harness name (`packages/core/src/plugin/agent.ts:12-13`).
### origin/v2 branch
- Base prompt: "You are an AI agent running in OpenCode, a coding agent harness." (`origin/v2:packages/core/src/session/runner/prompt/system.txt:1`) — harness identity without claiming to *be* OpenCode.
- `IdentityPlugin` splices `# Your Model / - Name / - Provider ID / - Model ID` as system part index 1 on `context` and `compaction` hooks (`origin/v2:packages/core/src/plugin/identity.ts:9-30`).

## Constants
| name | value | path:line |
|---|---|---|
| — | — | — |

## Evolution
- 2025-06-05 `35b03e4cb3` "claude oauth support": `anthropic_spoof.txt` "You are Claude Code, Anthropic's official CLI for Claude." prepended for Anthropic providers → [[provider-identity-shim]].
- 2025-08-11 `dac1506680` Claude Code sync dropped the "You are opencode" line from anthropic.txt; 2025-10-25 `795b845782` restored as "best coding agent on the planet".
- 2026-01-25 `94dd0a8dbe` "rm spoof": spoof moved out of core into the auth plugin.
- 2026-03-19 `1ac1a0287c` "anthropic legal requests (#18186)": bundled `opencode-anthropic-auth` plugin and the verbatim Claude Code prompt `anthropic-20250930.txt` removed; docs: "Anthropic explicitly prohibits this" (`git show 1ac1a0287c -- packages/web/src/content/docs/providers.mdx`).
- 2026-09-15 `d54fc305db` origin/v2: "powered by OpenCode" → "running in OpenCode, a coding agent harness" + identity plugin.

## Quirks / drift
- Legacy says both "You are OpenCode" and "You are powered by the model named X"; origin/v2 converges on pi's "operating inside <harness>" framing.

Contrast: pi says "operating inside pi, a coding agent harness" and never overrides model identity → [[pi--harness-identity|pi]], [[forced-model-identity-override]].
