---
type: concept
stage: model-interface
tier: candidate
aliases: [ThinkingLevel, thinkingLevelMap, clampThinkingLevel, ":high suffix", MIN_ANSWER_TOKENS, DEFAULT_THINKING_BUDGETS, thinkingFormat, thinking-format-adapter, adaptive-vs-budget-thinking, thinking-budget-answer-reserve, thinking-disable-fallback, thinking-level-suffix, forceAdaptiveThinking, thinkingTokenBudgetField]
harnesses: [pi]
---
A provider-neutral reasoning-effort scale (off/minimal/…/max), mapped for each provider and model onto the native control:
- an effort enum,
- an explicit token budget,
- an enable flag,
- chat-template kwargs.

The mapping models unsupported levels explicitly, clamps to the nearest supported level, sends an explicit "off" where the model allows it, and always reserves output tokens for the answer.

## Why
- Every server has its own reasoning knob:
  - `reasoning_effort`
  - `thinking{type, budget_tokens}`
  - `enable_thinking`
  - `chat_template_kwargs`
  - `reasoning{effort}`
  - Gemini `thinkingLevel` vs `thinkingBudget`
  - Mistral `prompt_mode`

  Hardcoded tables of model ids rot with every release ([[thinking-config-per-model-drift]]).
- "Off" can be encoded three ways: omit the field, send an explicit disable, or send a `none` effort. Each model accepts a different one, some models cannot disable thinking at all, and for some models the mode changes capabilities such as tools ([[thinking-off-not-honored]]).
- When reasoning and the answer share `max_tokens`, a reasoning-heavy turn can use the whole budget and produce no answer ([[thinking-consumes-answer-budget]]).

## Design space
- **Level vocabulary**
  - Raw provider values exposed to users.
  - A neutral scale plus a per-model map, where a missing key means the provider default and `null` means unsupported. *pi chose this.*
  - `xhigh`/`max` are opt-in only.
- **Clamping**
  - Search downward.
  - Search upward first, then downward. *pi chose this,* so a "minimal" request lands on "low" rather than "off"; the rationale is undocumented.
- **Control style**
  - Adaptive effort, where the model decides.
  - Explicit token budget with defaults of 1024/2048/8192/16384; `xhigh`/`max` clamp to `high` for budget-based providers.
  - Managed effort that can change mid-conversation.
- **Off encoding**
  - Omit the field.
  - Explicit disable. *pi chose this* (fbda78bfb, 6129971c0), gated on `off !== null`.
  - Lowest supported level with thoughts hidden, for models that cannot disable thinking. *pi chose this for Gemini 3:* d1613e3f5.
- **Where capability data lives**
  - Adapter-side substring tables. *pi used these at first.*
  - Generated catalog `thinkingLevelMap` and compat flags. *pi chose this:* 6184307c3, af2c35223, 16235fd93, dc84c1ac0.
- **Answer reserve**
  - None.
  - `MIN_ANSWER_TOKENS` always left for the answer, with the budget shrunk when the output cap is not larger than the budget. *pi chose this.*
- **User syntax**
  - A level suffix on the model reference (`model:high`), parsed at the last colon only after an exact match fails (see [[model-resolution]]).

## Implementations
- [[pi--thinking-level-abstraction|pi]] — `ThinkingLevel`, `thinkingLevelMap` and `clampThinkingLevel`. `simple-options.ts` holds the budgets and answer reserve. Per-adapter wire formats:
  - Anthropic: adaptive, budget and managed effort.
  - Bedrock: Claude and OpenAI-on-Bedrock.
  - Completions: 11 `thinkingFormat`s plus budget fields.
  - Responses and Codex: effort and summary.
  - Google: level vs budget.
  - Mistral: effort vs `prompt_mode`.

## Failures
- [[thinking-off-not-honored]]
- [[thinking-config-per-model-drift]]
- [[thinking-consumes-answer-budget]]

## Related
[[signed-reasoning-replay]] · [[max-tokens-context-clamp]] · [[model-catalog]] · [[model-resolution]] · [[virtual-model-router]] · [[cache-warming]]
