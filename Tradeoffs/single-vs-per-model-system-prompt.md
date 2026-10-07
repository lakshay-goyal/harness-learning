---
type: tradeoff
concepts: [minimal-system-prompt, per-model-system-prompt, harness-identity, dynamic-tool-guidelines]
harnesses: [pi, opencode]
---
# single-vs-per-model-system-prompt

**Axis**: one small harness prompt for every model, or a tuned base prompt per model family?

| option | pi | opencode | evidence |
|---|---|---|---|
| One prompt for all models | ✅ ~2 universal rules ("Be concise…", "Show file paths clearly…"); the rest contributed by tools/docs/context files | ❌ | pi: `packages/coding-agent/src/core/system-prompt.ts:122-123,152-191` → [[minimal-system-prompt]] |
| Separate prompt per family, chosen by model id substring | ❌ (Codex-specific instructions dropped: "use the default system prompt directly for Codex instructions", `4068bc556`) | ✅ `beast` (gpt-4/o1/o3), `gpt`, `codex`, `gpt-astra` (gpt-6), `gemini`, `anthropic`, `trinity`, `kimi`, `meta`, `default` | opencode: `packages/opencode/src/session/system.ts:28-51` → [[per-model-system-prompt]] |
| Agent prompt replaces base prompt | `--system-prompt` / SYSTEM.md replace preamble+tools+rules+docs | an agent with its own `prompt` replaces the provider prompt entirely | pi `packages/coding-agent/src/core/system-prompt.ts:199-205` → [[system-prompt-override]]; opencode `packages/opencode/src/session/llm/request.ts:60` |
| Identity | native model identity since `8b1cca827` ("You are actually not Claude, you are Pi" removed) | Claude Code spoof removed `94dd0a8dbe`; meta prompt templates `{{MODEL_NAME}}` | → [[harness-identity]], [[no-claude-subscription-auth]] |
| Prompt churn | 52 commits on `system-prompt.ts` | ~10 prompt files with their own histories (e.g. `da1d37274f` gpt prompt "modeled after codex cli"; `5cd8e68fdd` Astra ported from v2) | — |

**When each wins**
- **Single (pi)**: one text to keep cache-stable and consistent; behavior lives in tool descriptions that apply to every model. Works when models are strong generalists and the harness stays close to bash.
- **Per-model (opencode)**: models trained on a vendor's own harness (Codex, Claude Code) respond best to prompts shaped like their training distribution; also lets the toolset match ([[model-specific-toolset]]). Costs: N prompts to keep in sync, substring routing that silently misroutes new ids (e.g. Kimi by provider fix `91df883231`), and per-model regressions.

Related: [[minimal-system-prompt]] · [[minimal-vs-rich-toolset]] · [[prompt-cache-strategy]] · [[pi]] · [[opencode]] · [[Tradeoffs]]
