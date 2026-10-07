---
type: implementation
harness: pi
concept: thinking-level-abstraction
commit: b30a6dd77
files: [packages/ai/src/types.ts:89, packages/ai/src/types.ts:94, packages/ai/src/types.ts:831, packages/ai/src/models.ts:1222, packages/ai/src/models.ts:1235, packages/ai/src/api/simple-options.ts:24, packages/ai/src/api/simple-options.ts:67, packages/ai/src/api/anthropic-messages.ts:908, packages/ai/src/api/anthropic-messages.ts:1233, packages/ai/src/api/bedrock-converse-stream.ts:1254, packages/ai/src/api/openai-completions.ts:880, packages/ai/src/api/openai-responses.ts:363, packages/ai/src/api/openai-codex-responses.ts:582, packages/ai/src/api/google-shared.ts:48, packages/ai/src/api/google-generative-ai.ts:431, packages/ai/src/api/mistral-conversations.ts:186, packages/coding-agent/src/core/model-resolver.ts:204]
---
[[thinking-level-abstraction]] in [[pi]].

## Mechanism
- **Scale**: `ThinkingLevel = minimal|low|medium|high|xhigh|max`; `ModelThinkingLevel = off | ThinkingLevel` (packages/ai/src/types.ts:89-90). Coding-agent default `DEFAULT_THINKING_LEVEL = "medium"` (packages/coding-agent/src/core/defaults.ts:3-12). `max` added `fbdd46389` 2026-07-09.
- **Per-model map** `thinkingLevelMap: Partial<Record<level, string|null>>` → provider-native value; **missing = provider default, `null` = unsupported** (packages/ai/src/types.ts:91, 1140-1144). Populated by generator (`applyThinkingLevelMetadata` packages/ai/scripts/generate-models.ts:1038+; e.g. gpt-5* responses `off:null`, GPT-6 Astra/6.1 Sol reject `"none"`; Anthropic `max:"max"` opus/sonnet-4.6, `xhigh,max` opus-4.7+/sonnet-5, `off:null,xhigh,max` fable-5 :1092-1117; Copilot `minimal:"low"` checked against Copilot `/models` 2026-06-15 :513-520) → [[model-catalog]].
- `getSupportedThinkingLevels`: non-reasoning → `["off"]`; `null` levels excluded; **`xhigh`/`max` only if explicitly mapped** (packages/ai/src/models.ts:1222-1233).
- `clampThinkingLevel`: unsupported → search **upward first**, then downward (models.ts:1235-1254) — "minimal" on a minimal=null model yields "low", not "off".
- **Model reference suffix** `provider/id:level` parsed by `parseModelPattern`: full literal first (OpenRouter `x:exacto`), else split on last colon; invalid level warns in scope mode, fails in strict CLI mode (packages/coding-agent/src/core/model-resolver.ts:204-257; 9a7863fc9) → [[model-resolution]]. Unknown custom ids with `:level` force `reasoning:true` (:589-591; 1fc80f4f6).
- Sampling params merged `model.samplingParams` ⊕ `samplingParamsByThinkingLevel[clampedLevel]` ⊕ request (later wins) (packages/ai/src/api/simple-options.ts:24-34); only OpenAI-compatible adapters apply them (types.ts:206-213); applied LAST in completions payload (openai-completions.ts:1003-1007).
- **Token-budget providers** (simple-options.ts): `DEFAULT_THINKING_BUDGETS = {minimal:1024, low:2048, medium:8192, high:16384}` (:70-75); `clampReasoning` xhigh/max → high (:77-79); `MIN_ANSWER_TOKENS = 1024` always left for the answer (:67-68, 87-90); `adjustMaxTokensForThinking`: model cap if no caller cap else `min(base+budget, modelMax)`, shrink budget if `maxTokens <= budget` (:92-108) → [[thinking-consumes-answer-budget]], [[pi--max-tokens-context-clamp|max-tokens-context-clamp]].

**Per-provider wire mapping**

| adapter | on | off | notes |
|---|---|---|---|
| Anthropic (adaptive models, `forceAdaptiveThinking`) | `thinking:{type:"adaptive", display}` + `output_config.effort` (anthropic-messages.ts:1247-1252); effort = `thinkingLevelMap[level]` else minimal/low→low, medium, high, **anything else → "high"** (:908-926) | `{type:"disabled"}` iff `thinkingLevelMap.off !== null` (:1261-1263; 6129971c0 #2022, 9ccfcd7cf Fable 5) | adaptive list = opus-4.6/4.7/4.8/5, sonnet-4.6/5, fable-5, mythos-5 substring (generate-models.ts:626-643) |
| Anthropic budget | `{type:"enabled", budget_tokens: budget||1024, display}`; budget `min(adjusted, max(0, maxTokens−1024))` after context clamp (:961-975, 1253-1260) | as above | `interleaved-thinking-2025-05-14` beta only non-adaptive (:1108-1115; 9825c13f5); temperature dropped with thinking (:1194-1202) |
| Anthropic managed effort | always adaptive + `block_binding` + `effort:"high"`; per-turn markers (:1233-1241, 1518-1532) | impossible (`off:null` forced generate-models.ts:831) | → [[pi--signed-reasoning-replay\|signed-reasoning-replay]] |
| Bedrock Claude | adaptive (`supportsAdaptiveThinking` id+name normalized `[\s_.:]→-`, bedrock-converse-stream.ts:758-778) or budget `{1024,2048,8192,16384 ×(high/xhigh/max)}`, custom budgets only through high (:1281-1302); native `xhigh` if `supportsNativeXhighEffort` checked before map (:780-828) | — | interleaved beta non-adaptive (:1304-1306; d3d3ef415); GovCloud no display (454b9619c) |
| Bedrock OpenAI | `gpt-oss` flat `reasoning_effort` clamped low/medium/high; other `gpt-` nested `reasoning.effort`, minimal→low (:1311-1348; 2989eb581 #10142) | — | Bedrock rejects minimal |
| OpenAI Completions (`thinkingFormat`) | openai `reasoning_effort`; openrouter `reasoning:{effort}`; deepseek `thinking:{type:"enabled"}`+effort; together `reasoning:{enabled}`+effort; zai `thinking:{type:"enabled", clear_thinking:false}`+effort; qwen `enable_thinking`; qwen-chat-template `chat_template_kwargs{enable_thinking, preserve_thinking:true}`; chat-template `$var` placeholders `thinking.enabled\|effort\|budget`, `omitWhenOff`; baseten `chat_template_args`; string-thinking top-level `thinking:<string>`; ant-ling `reasoning:{effort}` only if mapped string (openai-completions.ts:880-977, 1034-1075; types.ts:94-102, 831-845) | per format: `thinking:{type:"disabled"}` (zai, deepseek unless off null), `enable_thinking:false`, `reasoning:{effort: off ?? "none"}`, `reasoning_effort = thinkingLevelMap.off` if string | detection order deepseek > zai > together > ant-ling > openrouter > openai (:1656-1666; DeepSeek URL match case-insensitive b647d1879 #7933); `supportsReasoningEffort` false for grok/zai/moonshot/together/CF gateway/nvidia/ant-ling (:1647-1648); budget field `thinking_token_budget` (vLLM) / `thinking_budget` (Qwen/SGLang) / `thinking_budget_tokens` (llama.cpp) clamped to leave ≥1024 (types.ts:104-105, 854-861; :979-985, 1012-1032; d07889da0, b23741269) |
| OpenAI Responses / Azure | `reasoning:{effort(mapped), summary: summary\|\|"auto"}` + include encrypted (openai-responses.ts:363-373); effort `"medium"` if only summary given | `reasoning:{effort: off ?? "none"}` unless off null; **Copilot omitted** (:374-378; bab58f821 #2567) | Azure same incl. off→"none" (azure-openai-responses.ts:224-240) |
| Codex | explicit effort mapped; `"none"` → `thinkingLevelMap.off` (undefined→"none", null→omit) | no effort & reasoning model & off≠null → `{effort: off ?? "none"}` (openai-codex-responses.ts:582-597; e86102f18 #9191) | |
| Google Gemini/Vertex | `usesGoogleThinkingLevel`: `/gemini-3(?:\.\d+)?-(?:pro\|flash)/`, `gemini-flash(-lite)?-latest`, `/gemma-?4/` → discrete `thinkingLevel`; else `thinkingBudget` (google-shared.ts:72-83); level = map value (case-insensitive) or pi level, must be minimal..high else throw (:48-65); `{includeThoughts:true, ...}` (google-generative-ai.ts:403-410) | budget models `{thinkingBudget:0}`; level models → `clampThinkingLevel(model,"off")` lowest level **without `includeThoughts`** (google-shared.ts:102-111; d1613e3f5 #2490 — Gemini 3.1 Pro can't disable, 3 Flash can't fully) ; no `reasoning` → explicit disabled (google-generative-ai.ts:319-321) | budgets 2.5-pro 128/2048/8192/32768, 2.5-flash-lite 512/2048/8192/24576, 2.5-flash 128/2048/8192/24576, other −1 dynamic (google-generative-ai.ts:431-471; b48d80293 #2838) |
| Mistral | `reasoning_effort = map[level] ?? "high"` if model has map; reasoning models without map (Magistral) → `prompt_mode:"reasoning"` | `map.off` if defined (e.g. "none") (mistral-conversations.ts:186-215; dc84c1ac0 #9678) | effort values none/low/medium/high/max (:34) |
| pi-messages | `reasoning` level sent to gateway (pi-messages.ts:370-402) | — | server-side mapping |

- Recorded per message: `thinkingLevel` (pi level requested by loop) and `providerThinkingLevel` (exact native effort) (packages/ai/src/types.ts:552-583; packages/agent/src/agent-loop.ts:408-409).
- Virtual models expose `thinkingLevels` (default `["off"]`), unoffered → `null` (packages/coding-agent/src/core/virtual-models.ts:84-102, 170-187); routed model clamps via `clampThinkingLevel` (model-runtime.ts:996-1027).
- Compaction passes `reasoning` only if `model.reasoning && thinkingLevel !== "off"` (d35935200 #1793).
- Model/thinking changes session-scoped unless persisted (2ff8ba622 #8356); cycling clamps per model (agent-session.ts:2518-2599).
- **Level resolution precedence (coding-agent)**: at session creation — explicit option → (resumed session) last `thinking_level_change` entry, else `defaultThinkingLevel` → per-model `modelThinkingLevels["provider/modelId"]` → `defaultThinkingLevel` → `"medium"` (`packages/coding-agent/src/core/sdk.ts:252-271`). On model switch — explicit level → per-model override → `defaultThinkingLevel` → current level → `"medium"` (`core/agent-session.ts:2667-2680`). Choosing a level in the interactive selector persists it per model (`setModelThinkingLevel`, `modes/interactive/interactive-mode.ts:4989`; `core/settings-manager.ts:908-915`). CLI shorthand `--model sonnet:high` / `--models sonnet:high,haiku:low` and `--thinking <level>` (`cli/args.ts:167`, help text `:302,388,397`). `hideThinkingBlock` (default false, `settings-manager.ts:1069-1071`) only hides blocks in the TUI; nothing changes on the wire.

## Constants
| name | value | path:line |
|---|---|---|
| DEFAULT_THINKING_BUDGETS | 1024/2048/8192/16384 | packages/ai/src/api/simple-options.ts:70-75 |
| MIN_ANSWER_TOKENS | 1024 | packages/ai/src/api/simple-options.ts:68 |
| Anthropic budget floor | `budget_tokens \|\| 1024`, `maxTokens − 1024` | packages/ai/src/api/anthropic-messages.ts:974, 1257 |
| Bedrock Claude budgets | +xhigh/max 16384 | packages/ai/src/api/bedrock-converse-stream.ts:1282-1289 |
| Gemini 2.5 budgets | see table | packages/ai/src/api/google-generative-ai.ts:441-466 |
| DEFAULT_THINKING_LEVEL | medium | packages/coding-agent/src/core/defaults.ts:3-12 |
| Anthropic display default | summarized | packages/ai/src/api/anthropic-messages.ts:259-271 |

## Evolution
- 2025-09-02 `004de3c9d` budget values 1024/2048/8192/16384 originate.
- 2025-12-15 `fbda78bfb` explicitly disable when reasoning undefined (#180).
- 2025-12-20 `36e17933d` Gemini 2.5 budgets (Cloud Code Assist).
- 2026-02-06 `d3d3ef415` Bedrock Opus 4.6 adaptive; 2026-02-26 `e9d0074fa` adaptive Sonnet 4.6 + clamp xhigh (#1548); 2026-02-27 `9825c13f5` no interleaved beta on adaptive, no temperature with thinking.
- 2026-02-28 `22b3be834` Z.ai `enable_thinking` (#1674).
- 2026-03-22 `d1613e3f5` explicit off across providers (#2490); `6129971c0` Anthropic disabled (#2022).
- 2026-03-24 `bab58f821` Copilot Responses no default reasoning (#2567).
- 2026-04-09 `b48d80293` flash-lite budget (#2861/#2838).
- 2026-04-16 `d1c6cb1e0` Opus 4.7 adaptive config (#3286); `acbf8eca0` thinkingDisplay.
- 2026-05-13 `e2b69a0bb` Mercury 2 `off:null`.
- 2026-05-22 `d801d88a1` adaptive for Anthropic-compatible aliases; 2026-06-10 `9ccfcd7cf` Fable 5 rejects disabled; 2026-06-23 `6184307c3` explicit compat metadata (id sniffing → generator).
- 2026-07-09 `fbdd46389` `max` level.
- 2026-08-05 `d07889da0` `thinking_token_budget` (#7638); 2026-08-18 `b23741269` generalized budget fields (#8275).
- 2026-08-17 `af2c35223` Google honors maps (#8135); 2026-09-17 `16235fd93` hardcoded Gemini tables removed (#9455).
- 2026-09-02 `4e69b0c28` managed effort; 2026-09-10 `e86102f18` Codex off; 2026-09-28 `dc84c1ac0` Mistral from map (#9678); 2026-10-06 `2989eb581` OpenAI on Bedrock.

## Evidence commits
004de3c9d · fbda78bfb · 36e17933d · d3d3ef415 · e9d0074fa · 9825c13f5 · 22b3be834 · d1613e3f5 · 6129971c0 · bab58f821 · b48d80293 · d1c6cb1e0 · acbf8eca0 · e2b69a0bb · d801d88a1 · 9ccfcd7cf · 6184307c3 · fbdd46389 · d07889da0 · b23741269 · af2c35223 · 16235fd93 · 4e69b0c28 · e86102f18 · dc84c1ac0 · 2989eb581 · 9a7863fc9 · 1fc80f4f6

## Quirks
- Clamp prefers the next **higher** level (more expensive) — rationale undocumented (open question).
- Anthropic `mapThinkingLevelToEffort` silently maps unmapped xhigh/max → "high", while Bedrock checks native xhigh first — divergence for custom models (unverified impact).
- Vertex budget table lacks the `2.5-flash-lite` row fixed in google.ts by b48d80293 → flash-lite on Vertex gets minimal=128 (google-vertex.ts:543); catalog membership unverified (data gitignored).
- Strict CLI `:level` refusal avoids "accidentally resolving to a different model" (model-resolver.ts:238-244).

## Failures
- [[thinking-off-not-honored]] · [[thinking-config-per-model-drift]] · [[thinking-consumes-answer-budget]] · [[endpoint-rejects-request-field]] · [[stale-thinking-signature-after-prefix-change]]
