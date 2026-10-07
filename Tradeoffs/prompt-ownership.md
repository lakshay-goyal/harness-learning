---
type: tradeoff
concepts: [minimal-system-prompt, per-model-system-prompt, harness-identity, dynamic-tool-guidelines, personality-variants, system-prompt-override, message-role-layering]
---
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
