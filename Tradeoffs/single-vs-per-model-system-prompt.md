---
type: tradeoff
concepts: [minimal-system-prompt, per-model-system-prompt, harness-identity, dynamic-tool-guidelines, personality-variants, system-prompt-override, message-role-layering]
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

## Also: [[codex]] (folded from `prompt-ownership`)
Who owns the base system prompt: the harness (one model-agnostic text in code) or the model vendor (per-model text shipped as catalog data)?

| dimension | pi — harness-owned minimal prompt | codex — model-catalog-owned per-model prompt |
|---|---|---|
| Where the text lives | `packages/coding-agent/src/core/system-prompt.ts` sections ([[pi--minimal-system-prompt]]) | `model_messages.instructions_template` in bundled/remote `codex-rs/models-manager/models.json`; fallback `codex-rs/models-manager/prompt.md` ([[codex--per-model-system-prompt]]) |
| Size | ~680 tokens default | 17–22 KB per model (~5k tokens); ~7 KB for codex-tuned models (`916fdc2a37`) |
| Who can change it | harness release; user SYSTEM.md / APPEND / hook ([[pi--system-prompt-override]]) | vendor via remote catalog without client release; user `base_instructions` / `model_instructions_file` "STRONGLY DISCOURAGED" ([[codex--system-prompt-override]]) |
| Model identity | never overridden; harness identity only ([[pi--harness-identity]], [[forced-model-identity-override]]) | names the model and gives it a persona ("You are GPT-5.1 running in the Codex CLI") ([[codex--harness-identity]]) |
| Tool guidance | generated from declared tools (`promptSnippet`) ([[pi--dynamic-tool-guidelines]]) | fixed document; sections for disabled tools stripped by heading match ([[codex--dynamic-tool-guidelines]]) |
| Volatile policy | section patches appended as system deltas ([[transcript-carried-system-prompt]]) | developer-role fragments diffed per request ([[codex--message-role-layering]], [[codex--world-state-diff-injection]]) |
| Persona | none | user-selectable 2026-01 → retired 2026-09; one voice per model ([[codex--personality-variants]]) |
| Typical failures | rules naming absent tools ([[prompt-names-unavailable-tools]]), identity override reverted | stale limits in prose ([[prompt-states-stale-harness-limits]]), rewrites dropping lines ([[prompt-rewrite-drops-load-bearing-lines]]), anchor splices failing ([[anchor-based-prompt-injection-silently-fails]]), per-generation bias-to-action patches ([[premature-turn-end]]) |
| Prompt files as code | TS template | Markdown/JSON data; formatter corrupted a grammar ([[formatter-corrupts-prompt-file]]); orphaned per-model files after `a1abd53b6a` |

**When harness-owned wins** — many third-party models, no control over training; prompt must be small, model-agnostic, cache-cheap, and auditable in one place; extension authors contribute sections.

**When model-owned wins** — the vendor ships both harness and models, trains models on the harness, and iterates prompts per model generation (eval-driven, 16 plan-mode prompt commits in two weeks); remote updates fix behaviour without a client release. Cost: prompt drifts from harness code, needs per-line provenance and a fallback for unknown models.
