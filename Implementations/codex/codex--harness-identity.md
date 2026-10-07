---
type: implementation
harness: codex
concept: harness-identity
commit: 622e9e3696
files: [codex-rs/protocol/src/prompts/base_instructions/default.md:1, codex-rs/protocol/src/prompts/base_instructions/default.md:9, codex-rs/core/gpt_5_1_prompt.md:1, codex-rs/core/gpt_5_codex_prompt.md:1, codex-rs/models-manager/models.json]
---
[[harness-identity]] in [[codex]].

## Mechanism
- Fallback/base: "You are a coding agent running in the Codex CLI, a terminal-based coding assistant… Within this context, Codex refers to the open-source agentic coding interface (not the old Codex language model built by OpenAI)." (`codex-rs/protocol/src/prompts/base_instructions/default.md:1`, `:9`).
- Per-model prompts name the model: "You are GPT-5.1 running in the Codex CLI" (`codex-rs/core/gpt_5_1_prompt.md:1`), "You are Codex, based on GPT-5." (`codex-rs/core/gpt_5_codex_prompt.md:1`), "You are Codex, an agent based on GPT-6." (`codex-rs/models-manager/models.json`, gpt-6-astra). Possible because the prompt is per-model catalog data → [[codex--per-model-system-prompt]].
- Newest catalog prompts add persona: "You have a vivid inner life as Codex: intelligent, playful, curious, and deeply present." (gpt-5.5, `c10f95ddac` 2026-04-24) → [[codex--personality-variants]].
- Surface disambiguation is part of identity: the CLI prompt forbids Codex-web/ChatGPT conventions (citation syntax "【F:README.md†L5-L14】", container assumptions) → [[foreign-harness-tool-hallucination]].

## Evolution
- 2025-04-16 `59a180ddec` TS original (`75febbdefa:codex-cli/src/utils/agent/agent-loop.ts:977`): "Don't confuse yourself with the old Codex language model built by OpenAI many moons ago (this is understandably top of mind for you!)".
- 2025-04-24 `31d0d7a305` Rust prompt inherited the Codex-web container identity ("You are a deployed coding agent. Your session is backed by a container..."); replaced 2025-08-05 `d31e149cb1` / 2025-08-07 `81b148bda2` by the CLI identity.
- 2025-09-14 `916fdc2a37` "You are Codex, based on GPT-5." (codex-tuned prompt); 2025-11-13 `8dcbd29edd` "You are GPT-5.1 running in the Codex CLI".
- 2026-04-24 `c10f95ddac` persona text in catalog prompts.

## Versus pi
- [[pi--harness-identity]] states only harness identity and must not override a third-party model's identity ([[forced-model-identity-override]]); codex, serving its own models with per-model prompts, names the model and its persona.
