---
type: constants
harnesses: [pi]
---
# Constants

Cross-harness hardcoded values. One row per constant (some rows bundle 2–5 sibling values that share a file/policy); one value+location column pair per harness. New harness → add `<harness>` + `<harness> location` columns; `—` where absent.

- **pi** @ `b30a6dd77` (2026-10-07), [[pi]]. Source census: `git grep` of UPPER_CASE numerics (395 hits minus tests/examples/easter eggs), `??`/`||` defaults, `setTimeout(…, N)`, `git log -S/-G` for history. Per-model rows of `packages/ai/src/models.generated.ts` skipped; generator-wide defaults kept. 238 rows (~300 distinct values).
- Structural fact: `packages/agent/src` (the core loop) has **zero numeric literals ≥ 10** — no turn cap, step limit or loop timeout (`grep -rnE '[0-9]{2,}' packages/agent/src` → empty; `maxTurns|maxSteps|maxIterations` → no hits in `packages/*/src`). Every limit lives at the edges: provider, tools, compaction, TUI. See [[no-turn-cap]].
- Paths are repo-relative to the pi repo. Values in backticks are literal source.

## 01-loop

| constant (neutral) | pi | pi location | tradeoff | history |
|---|---|---|---|---|
| Max agent turns per prompt · [[turn-loop]] · [[no-turn-cap]] | none (absent) | `packages/agent/src/agent-loop.ts` (no numeric literals) | Trust the model / user Esc instead of hard stop; risk runaway loops | deliberate absence (no `maxTurns` anywhere) |
| Agent-level retry enabled · [[auto-retry-backoff]] | `retry.enabled` = `true` | `packages/coding-agent/src/core/settings-manager.ts:1011` | Auto-recover transient 429/5xx/overloaded vs surfacing errors | bb445d24f (2025-12-10) introduced |
| Agent-level max retries · [[auto-retry-backoff]] | `retry.maxRetries` = `3` | `packages/coding-agent/src/core/settings-manager.ts:1026` | Bounded attempts; initial call not counted | bb445d24f: "exponential backoff (2s, 4s, 8s)", fixes #157; unchanged since |
| Agent-level backoff base · [[auto-retry-backoff]] | `retry.baseDelayMs` = `2000` ms | `packages/coding-agent/src/core/settings-manager.ts:1027` | `base * 2^(n-1)` → 2s/4s/8s | bb445d24f introduced; unchanged |
| Agent-level backoff cap · [[auto-retry-backoff]] | `DEFAULT_MAX_AGENT_RETRY_DELAY_MS` = `60_000` ms | `packages/ai/src/utils/retry.ts:126` | Cap per-attempt sleep so large `maxRetries` doesn't sleep hours | moved to ai pkg to share with SDK (comment `packages/ai/src/utils/retry.ts:110-115`) |
| Backoff formula · [[auto-retry-backoff]] | `retryDelayMs()` = `base * 2 ** (attempt-1)`, min(cap) | `packages/ai/src/utils/retry.ts:128-131` | No jitter at agent level (jitter only at provider level) | — |
| Steering queue delivery · [[steering-queue]] | `steeringMode` = `"one-at-a-time"` | `packages/coding-agent/src/core/settings-manager.ts:854` | Mid-run user messages injected one per turn, not batched | 0119d7610 (2025-12-09, AgentSession queue mode) |
| Follow-up queue delivery · [[follow-up-queue]] | `followUpMode` = `"one-at-a-time"` | `packages/coding-agent/src/core/settings-manager.ts:864` | Same as above for post-run queue | — |
| Default thinking level · [[thinking-level-abstraction]] | `DEFAULT_THINKING_LEVEL` = `"medium"` | `packages/coding-agent/src/core/defaults.ts:3` | Moderate reasoning cost by default | — |
| Default tool set · [[minimal-default-toolset]] | `DEFAULT_TOOL_NAMES` = `read, bash, edit, write` | `packages/coding-agent/src/core/settings-manager.ts:215` | Minimal 4-tool surface; grep/find/ls opt-in | — |
| Durable harness retry policy · [[auto-retry-backoff]] · [[durable-execution]] | `DEFAULT_RETRY_POLICY` = `{3, 2000ms, 60000ms}` | `packages/durable/src/harness/agent.ts:19-24` | Mirrors coding-agent defaults in new durable harness | ed0d6b91b-era durable packages (2026-09) |
| Durable progress emit interval · [[durable-execution]] | `DEFAULT_PROGRESS_POLICY` = `partial 100ms / output 100ms` | `packages/durable/src/harness/agent.ts:33-36` | Throttle persisted streaming deltas | — |
| Durable generation poll · [[durable-execution]] · [[deferred-responses]] | `DEFAULT_POLL_AFTER_MS` = `5000` ms | `packages/durable/src/harness/generation.ts:109` | Poll cadence for async generation | (unverified purpose detail) |
| Process kill escalation · [[process-tree-kill]] | SIGTERM → SIGKILL after `5000` ms | `packages/coding-agent/src/core/exec.ts:61` | Grace for cleanup vs hung children | — |
| Child stdio drain after exit · [[shell-execution]] | `EXIT_STDIO_GRACE_MS` = `100` ms | `packages/coding-agent/src/utils/child-process.ts:16` | Catch trailing output vs exit latency | — |
| RPC client wait defaults · [[headless-rpc-mode]] | `waitForIdle/collectEvents/promptAndWait` = `60000` ms | `packages/coding-agent/src/modes/rpc/rpc-client.ts:471,491,513` | Test/SDK convenience timeouts | — |

## 02-model-interface

| constant (neutral) | pi | pi location | tradeoff | history |
|---|---|---|---|---|
| Provider (SDK) retries · [[auto-retry-backoff]] | `retry.provider.maxRetries` / `options.maxRetries ?? 0` = `0` | `packages/ai/src/utils/provider-retry.ts:111` | SDK retries disabled; app-level retry instead (abortable, visible) | bb445d24f set Anthropic SDK `maxRetries: 0`; 8fb1e877c (2026-05-26) "disable hidden provider 429 retries (#4991)" |
| Provider backoff · [[auto-retry-backoff]] | `getRetryDelayMs` = `min(0.5*2^n, 8)s × (1 - rand*0.25)` | `packages/ai/src/utils/provider-retry.ts:67-68` | Mirrors OpenAI/Anthropic SDK policy, made abortable | — |
| Max server-requested retry delay · [[auto-retry-backoff]] | `DEFAULT_MAX_RETRY_DELAY_MS` / `retry.provider.maxRetryDelayMs` = `60_000` ms | `packages/ai/src/utils/provider-retry.ts:1`; `packages/coding-agent/src/core/settings-manager.ts:1061` | Fail fast (and show user) instead of silently honoring hours-long `retry-after` | 030a61d88 (2026-02-01) "Gemini CLI requested hours"; renamed/migrated c06750410 (settings migration `packages/coding-agent/src/core/settings-manager.ts:561`) |
| Retryable statuses · [[auto-retry-backoff]] | `isRetryableProviderError` = 408, 409, 429, ≥500, `x-should-retry` | `packages/ai/src/utils/provider-retry.ts:25-37` | SDK parity | — |
| HTTP idle timeout · [[http-transport-hardening]] | `DEFAULT_HTTP_IDLE_TIMEOUT_MS` = `300_000` ms (choices 30s/1m/2m/5m/off) | `packages/coding-agent/src/core/http-dispatcher.ts:4,8-14` | Long thinking pauses vs dead-socket detection | 849f9d9c5 (2026-05-20, #4759) |
| Happy-eyeballs attempt timeout · [[http-transport-hardening]] | `DEFAULT_AUTO_SELECT_FAMILY_ATTEMPT_TIMEOUT_MS` = `2_000` ms | `packages/coding-agent/src/core/http-dispatcher.ts:6` | Node's 250ms default kills high-latency connects | — |
| WebSocket connect timeout · [[http-transport-hardening]] | `DEFAULT_WEBSOCKET_CONNECT_TIMEOUT_MS` = `15_000` ms | `packages/ai/src/api/openai-codex-responses.ts:57` | Handshake bound | — |
| Codex retries · [[auto-retry-backoff]] | `DEFAULT_MAX_RETRIES` / `BASE_DELAY_MS` = `0` / `1000` ms | `packages/ai/src/api/openai-codex-responses.ts:54-55` | Same "no hidden retries" stance | 8fb1e877c |
| Codex WS session cache TTL · [[http-transport-hardening]] · [[session-affinity-cache-routing]] | `SESSION_WEBSOCKET_CACHE_TTL_MS` = `5 * 60 * 1000` | `packages/ai/src/api/openai-codex-responses.ts:866` | Reuse WS (server-side cached context) for 5 min idle | a26a9cfab (2026-02-13) |
| Codex WS max age · [[http-transport-hardening]] | `SESSION_WEBSOCKET_MAX_AGE_MS` = `55 * 60 * 1000` | `packages/ai/src/api/openai-codex-responses.ts:867` | Rotate before server's ~60 min limit | 23d146261 (2026-07-03) "rotate stale Codex websocket sessions", #6268 |
| Codex request compression · [[http-transport-hardening]] | `REQUEST_COMPRESSION_ZSTD_LEVEL` = `3` | `packages/ai/src/api/openai-codex-responses.ts:60` | CPU vs upload size (matches Codex client) | — |
| Mistral request timeout · [[http-transport-hardening]] | `60_000` ms | `packages/ai/src/api/mistral-conversations.ts:307` | Only provider with hard default timeout | — |
| Mistral tool-call id length · [[tool-call-id-normalization]] | `MISTRAL_TOOL_CALL_ID_LENGTH` = `9` | `packages/ai/src/api/mistral-conversations.ts:28` | Provider requires 9-char ids | — |
| Provider error body cap · [[errors-as-stream-events]] | `MAX_PROVIDER_ERROR_BODY_CHARS` = `4000` | `packages/ai/src/utils/error-body.ts:16` | Useful diagnostics vs flooding transcript | — |
| Bedrock diagnostic value cap · [[errors-as-stream-events]] | `MAX_BEDROCK_DIAGNOSTIC_VALUE_CHARS` = `200` | `packages/ai/src/api/bedrock-converse-stream.ts:415` | — | — |
| Diagnostic string cap · [[errors-as-stream-events]] | `maxLength` = `8192` | `packages/ai/src/api/pi-messages.ts:121` | — | — |
| Default context window (custom models) · [[model-catalog]] | `definition.contextWindow ?? 128000` = `128000` | `packages/coding-agent/src/core/provider-composer.ts:242` | Safe-ish assumption for unknown models | ef6af5ebb (2026-03-29) → 9993c9690 (2026-07-14 model runtime) |
| Default max output tokens (custom models) · [[model-catalog]] | `definition.maxTokens ?? 16384` = `16384` | `packages/coding-agent/src/core/provider-composer.ts:243` | — | same |
| Generator fallback context/max tokens · [[model-catalog]] | `limit?.context \|\| 4096` = `4096` | `packages/ai/scripts/generate-models.ts:1494-1495` (and ~15 sites) | Pessimistic when models.dev lacks data | — |
| llama.cpp fallback context · [[model-catalog]] | `128000` | `packages/coding-agent/src/extensions/llama/provider.ts:78` | — | — |
| Context safety margin for max_tokens clamp · [[max-tokens-context-clamp]] | `CONTEXT_SAFETY_TOKENS` = `4096` | `packages/ai/src/api/simple-options.ts:15` | `maxTokens = min(req, window - estimate - 4096)`; absorbs estimate error | 09f105957 (2026-06-25) "clamp streamSimple max tokens" #5595/#6061 |
| Minimum max_tokens · [[max-tokens-context-clamp]] | `MIN_MAX_TOKENS` = `1` | `packages/ai/src/api/simple-options.ts:16` | Never send 0/negative | 09f105957 |
| Answer room under thinking budget · [[thinking-level-abstraction]] | `MIN_ANSWER_TOKENS` = `1024` | `packages/ai/src/api/simple-options.ts:68` | Thinking can't eat whole response ceiling | d07889da0 (2026-08-05), b23741269 |
| Thinking budgets · [[thinking-level-abstraction]] | `DEFAULT_THINKING_BUDGETS` = minimal 1024 / low 2048 / medium 8192 / high 16384 | `packages/ai/src/api/simple-options.ts:70-75` | xhigh/max clamp to high | values since 004de3c9d (2025-09-02); const named b23741269 |
| Bedrock Claude budgets · [[thinking-level-abstraction]] | `defaultBudgets` = same + xhigh/max 16384 | `packages/ai/src/api/bedrock-converse-stream.ts:1282-1289` | — | fd268479a |
| Anthropic budget floor · [[thinking-level-abstraction]] | `budget_tokens \|\| 1024`, `maxTokens - 1024` | `packages/ai/src/api/anthropic-messages.ts:974,1257` | Anthropic minimum budget is 1024 | — |
| Gemini 2.5 budgets · [[thinking-level-abstraction]] | pro: 128/2048/8192/32768; flash: 128/2048/8192/24576; flash-lite min 512 | `packages/ai/src/api/google-generative-ai.ts:441-466` | Model-specific minima | 36e17933d |
| OpenAI min output tokens · [[max-tokens-context-clamp]] | `OPENAI_RESPONSES_MIN_OUTPUT_TOKENS` = `16` | `packages/ai/src/api/openai-responses.ts:33`; `packages/ai/src/api/azure-openai-responses.ts:19` | API rejects <16 | — |
| OpenAI prompt cache key length · [[session-affinity-cache-routing]] | `OPENAI_PROMPT_CACHE_KEY_MAX_LENGTH` = `64` | `packages/ai/src/api/openai-prompt-cache.ts:1` | API limit | — |
| OpenAI long-context pricing threshold · [[usage-cost-accounting]] | `OPENAI_LONG_CONTEXT_INPUT_THRESHOLD` = `272000` | `packages/ai/scripts/generate-models.ts:395` | 2x input / 1.5x output above | — |
| Codex context windows · [[model-catalog]] | `CODEX_CONTEXT` / `CODEX_SPARK_CONTEXT` / `CODEX_MAX_TOKENS` = 272000 / 128000 / 128000 | `packages/ai/scripts/generate-models.ts:3261-3264` | — | — |
| OpenAI decisions max images · [[structured-classifier-api]] | `MAX_IMAGES` = `128` | `packages/ai/src/api/openai-decisions.ts:33` | — | — |
| Image request limits (provider) · [[image-normalization]] | `applyImageInputMetadata` = Anthropic 32 MiB req, 100 imgs (200k ctx) else 600; Bedrock 20/msg; OpenAI 512 MiB, 1500; Google 20 MiB, 3600 | `packages/ai/scripts/generate-models.ts:997-1009` | Encode provider caps as model metadata | f5c946480 (2026-09-20, #9631) |
| Remote model catalog refresh · [[model-catalog]] | `REMOTE_CATALOG_REFRESH_INTERVAL_MS` = `4h` | `packages/coding-agent/src/core/remote-catalog-provider.ts:15` | Freshness vs network | — |
| Remote catalog attempt timeout · [[model-catalog]] | `REMOTE_CATALOG_ATTEMPT_TIMEOUT_MS` = `4_000` ms | `packages/coding-agent/src/core/remote-catalog-provider.ts:14` | Startup not blocked | — |
| Catalog refresh abort (interactive/RPC) · [[model-catalog]] | `15_000` ms | `packages/coding-agent/src/main.ts:941`; `packages/coding-agent/src/modes/interactive/interactive-mode.ts:1143` | — | — |
| OAuth min validity before use · [[subscription-oauth-auth]] | `DEFAULT_OAUTH_MINIMUM_VALIDITY_MS` = `5 min` | `packages/ai/src/auth/resolve.ts:102` | Refresh early to avoid mid-request expiry | 99e34013d (2026-07-27) |
| OAuth refresh timeout · [[subscription-oauth-auth]] | `DEFAULT_OAUTH_REFRESH_TIMEOUT_MS` = `15_000` | `packages/ai/src/auth/resolve.ts:103` | — | — |
| Anthropic OAuth callback port · [[subscription-oauth-auth]] | `CALLBACK_PORT` = `53692` | `packages/ai/src/auth/oauth/anthropic.ts:20` | Fixed registered redirect | 92882dc4c (2026-03-13, #2119) |
| OpenAI ChatGPT OAuth callback port · [[subscription-oauth-auth]] | `CALLBACK_PORT` = `1455` | `packages/ai/src/auth/oauth/openai-chatgpt.ts:23` | Same port as Codex CLI | 02eed88fd (2026-09-29) |
| Radius OAuth callback port · [[subscription-oauth-auth]] | `CALLBACK_PORT` = `1456` | `packages/ai/src/auth/oauth/radius.ts:19` | — | — |
| ChatGPT token expiry margin · [[subscription-oauth-auth]] | `EXPIRY_MARGIN_MS` = `3 min` | `packages/ai/src/auth/oauth/openai-chatgpt.ts:29` | — | — |
| xAI refresh skew / default lifetime · [[subscription-oauth-auth]] | `REFRESH_SKEW_MS` / `DEFAULT_TOKEN_LIFETIME_SECONDS` = 5 min / 3600 s | `packages/ai/src/auth/oauth/xai.ts:13-14` | — | — |
| Radius token skew · [[subscription-oauth-auth]] | `TOKEN_EXPIRY_SKEW_MS` = `60_000` | `packages/ai/src/auth/oauth/radius.ts:22` | — | — |
| Device-code flow · [[subscription-oauth-auth]] | `MINIMUM_INTERVAL_MS` / `DEFAULT_POLL_INTERVAL_SECONDS` / `SLOW_DOWN_INTERVAL_INCREMENT_MS` = 1000 / 5 / 5000 | `packages/ai/src/auth/oauth/device-code.ts:5,7,9` | RFC 8628 compliance | — |
| Device-code timeout (Codex, Kimi) · [[subscription-oauth-auth]] | `DEVICE_CODE_TIMEOUT_SECONDS` = `15*60` | `packages/ai/src/auth/oauth/openai-codex.ts:31`; `packages/ai/src/auth/oauth/kimi-coding.ts:16` | — | — |
| Kimi refresh retries / request timeout · [[subscription-oauth-auth]] | `REFRESH_MAX_RETRIES` / `REQUEST_TIMEOUT_MS` = 3 / 30s | `packages/ai/src/auth/oauth/kimi-coding.ts:18-19` | — | — |
| Meta API key lifetime · [[subscription-oauth-auth]] | `API_KEY_LIFETIME_MS` = `24h` | `packages/ai/src/auth/oauth/meta.ts:25` | — | — |
| OpenRouter login / exchange timeout · [[subscription-oauth-auth]] | `LOGIN_TIMEOUT_MS` / `TOKEN_EXCHANGE_TIMEOUT_MS` = 5 min / 30s | `packages/ai/src/auth/oauth/openrouter.ts:21-22` | — | — |
| Copilot token retry · [[subscription-oauth-auth]] | `500 * 2 ** retry`; `{maxRetries: 2, maxElapsedMs: 5000}` | `packages/ai/src/auth/oauth/github-copilot.ts:155,398` | — | — |
| Bearer token min expiry for `auth print` · [[credential-resolution]] | `DEFAULT_BEARER_TOKEN_MIN_EXPIRY_MS` = `30 min` | `packages/coding-agent/src/cli/credential-print.ts:7` | External tools get usable token | 99e34013d |
| llama.cpp classify readout · [[structured-classifier-api]] | `MIN_READOUT_DEPTH` / `READOUT_DEPTH_PER_LABEL` / `READOUT_ESCALATION` = 256 / 16 / [4096, 32768] | `packages/ai/src/api/llama-cpp-classify.ts:44-47` | — | — |
| Faux provider token chunking · [[harness-evals]] | `DEFAULT_MIN/MAX_TOKEN_SIZE` = 3 / 5 | `packages/ai/src/providers/faux.ts:29-30` | Test provider | — |

## 03-tools

| constant (neutral) | pi | pi location | tradeoff | history |
|---|---|---|---|---|
| Tool output max lines · [[tool-output-truncation]] | `DEFAULT_MAX_LINES` = `2000` | `packages/coding-agent/src/core/tools/truncate.ts:11` | Whichever of lines/bytes hits first | de77cd141 (2025-12-07) introduced |
| Tool output max bytes · [[tool-output-truncation]] | `DEFAULT_MAX_BYTES` = `50 * 1024` | `packages/coding-agent/src/core/tools/truncate.ts:12` | Context cost vs needing re-reads | de77cd141: **30KB** → 306f9cc66 (same day, #134): **50KB** |
| grep line length cap · [[tool-output-truncation]] | `GREP_MAX_LINE_LENGTH` = `500` chars | `packages/coding-agent/src/core/tools/truncate.ts:13` | Minified files blow budget | b813a8b92 (2025-12-07, #134) |
| read truncation direction · [[tool-output-truncation]] | (head) = head | `packages/coding-agent/src/core/tools/read.ts:96` | Offset/limit continuation message | de77cd141 |
| bash truncation direction · [[tool-output-truncation]] | (tail) = last 2000 lines/50KB; full output to temp file | `packages/coding-agent/src/core/tools/bash.ts:255` | Errors are at the end | de77cd141 |
| bash default timeout · [[shell-execution]] · [[no-bash-default-timeout]] | **none** | `packages/coding-agent/src/core/tools/bash.ts:42` | Long builds allowed; model must opt in | 29900ce64 (2025-11-12): **30s → none** ("commands run until completion unless specified") |
| bash max timeout · [[shell-execution]] · [[no-bash-default-timeout]] | `MAX_TIMEOUT_MS` = `2_147_483_647` ms (int32 setTimeout max) | `packages/coding-agent/src/core/tools/bash.ts:22` | Node timer overflow guard | — |
| bash structured output (codemode) · [[structured-tool-output]] | `STRUCTURED_OUTPUT_MAX_BYTES` = `1024 * 1024` | `packages/coding-agent/src/core/tools/bash.ts:24` | Scripts get 20× more than model | 1ff5b6fdd (2026-09-29) |
| bash render throttle · [[differential-tui-rendering]] | `BASH_UPDATE_THROTTLE_MS` = `100` ms | `packages/coding-agent/src/core/tools/renderers/bash.ts:19` | — | — |
| bash preview lines (collapsed) · [[differential-tui-rendering]] | `BASH_PREVIEW_LINES` = `5` | `packages/coding-agent/src/core/tools/renderers/bash.ts:18` | — | — |
| `!` command preview lines · [[differential-tui-rendering]] | `PREVIEW_LINES` = `20` | `packages/coding-agent/src/modes/interactive/components/bash-execution.ts:19` | — | — |
| find result limit · [[search-tools]] | `DEFAULT_LIMIT` = `1000` | `packages/coding-agent/src/core/tools/find.ts:41` | — | de77cd141 |
| grep match limit · [[search-tools]] | `DEFAULT_LIMIT` = `100` | `packages/coding-agent/src/core/tools/grep.ts:41` | — | de77cd141 |
| ls entry limit · [[search-tools]] | `DEFAULT_LIMIT` = `500` | `packages/coding-agent/src/core/tools/ls.ts:23` | — | de77cd141 |
| Collapsed tool args preview · [[differential-tui-rendering]] | `COLLAPSED_ARGS_CHARS` = `100` (80 in codemode) | `packages/coding-agent/src/core/tools/render-utils.ts:71`; `packages/coding-agent/src/extensions/codemode/renderer.ts:21` | — | — |
| Fallback tool preview · [[differential-tui-rendering]] | `FALLBACK_PREVIEW_LINES` = `10` | `packages/coding-agent/src/modes/interactive/components/tool-execution.ts:23` | — | — |
| Write highlight cutoff · [[differential-tui-rendering]] | `WRITE_PARTIAL_FULL_HIGHLIGHT_LINES` = `50` | `packages/coding-agent/src/core/tools/renderers/write.ts:29` | — | — |
| Image max dimension · [[image-normalization]] | `DEFAULT_OPTIONS.maxWidth/maxHeight` = `2000 × 2000` | `packages/coding-agent/src/utils/image-resize-core.ts:35-36` | Model compatibility (many-image Anthropic limit is 2000px) | 4a32af253 (2026-01-02) |
| Image max encoded bytes · [[image-normalization]] | `DEFAULT_MAX_BYTES` = `4.5 * 1024 * 1024` | `packages/coding-agent/src/utils/image-resize-core.ts:32` | Headroom under Anthropic's 5MB base64 limit | 69dc6b078 (2026-01-03, #424) |
| JPEG quality ladder · [[image-normalization]] | `jpegQuality` / `qualitySteps` = 80 then 85,70,55,40 | `packages/coding-agent/src/utils/image-resize-core.ts:38,132` | — | 69dc6b078 |
| Generated image resize default · [[image-normalization]] | `DEFAULT_IMAGE_RESIZE` = 2000/2000/4.5MiB/80 | `packages/ai/scripts/generate-models.ts:424-429` | "cache-safe" historical default for unknown providers | f5c946480 (2026-09-20) |
| Image type sniff bytes · [[image-normalization]] | `IMAGE_TYPE_SNIFF_BYTES` = `4100` | `packages/coding-agent/src/utils/mime.ts:3` | file-type lib requirement | — |
| MCP model-facing output cap · [[mcp-integration]] | `MCP_OUTPUT_MAX_BYTES` = `20 * 1024` (middle cut) | `packages/coding-agent/src/extensions/mcp/tools.ts:51` | Smaller than built-in 50KB | 8562bcf66 (2026-09-29) |
| MCP tool name length · [[mcp-integration]] | `MAX_TOOL_NAME_LENGTH` = `64` | `packages/coding-agent/src/extensions/mcp/tools.ts:49` | Provider name rule | 8562bcf66 |
| MCP per-call timeout · [[mcp-integration]] | `DEFAULT_TIMEOUT_SECONDS` = `60` s | `packages/coding-agent/src/extensions/mcp/runtime.ts:48` | Per-server overridable | 8562bcf66 |
| MCP stderr tail · [[mcp-integration]] | `STDERR_TAIL_CHARS` = `2_000` | `packages/coding-agent/src/extensions/mcp/runtime.ts:49` | — | — |
| MCP HTTP connect retry delays · [[mcp-integration]] · [[auto-retry-backoff]] | `CONNECT_RETRY_DELAYS_MS` = `[250, 1_000]` | `packages/coding-agent/src/extensions/mcp/runtime.ts:51` | 2 quick retries, only for URL servers | 8562bcf66 |
| tool_search default results · [[deferred-tool-loading]] | `DEFAULT_TOOL_SEARCH_LIMIT` = `8` | `packages/coding-agent/src/extensions/tool-search/tool.ts:21` | — | 8562bcf66 |
| Nested tool-call record limits · [[nested-tool-calls]] | `NESTED_CALL_LIMITS` = 256 calls, 8 KiB/call args, 32 KiB total, 500 error chars | `packages/coding-agent/src/core/nested-tool-calls.ts:25-30` | Bound transcript bloat from codemode | 8562bcf66 |
| codemode script timeout · [[code-mode]] | `DEFAULT_TIMEOUT_MS` = `300_000` | `packages/codemode/src/runtime/host.ts:22` | — | — |
| codemode heap · [[code-mode]] | `CODEMODE_MEMORY_LIMIT_BYTES` = `256 MiB` | `packages/coding-agent/src/extensions/codemode/execute.ts:56` | QuickJS shares process; else wasm 4 GiB | 8562bcf66 |
| codemode output budget · [[code-mode]] | `DEFAULT_MAX_OUTPUT_TOKENS` = `10_000` tokens (chars/4) | `packages/coding-agent/src/extensions/codemode/execute.ts:246,248` | — | 8562bcf66 |
| codemode concurrent model calls · [[code-mode]] | `MAX_CONCURRENT_MODEL_CALLS` = `4` | `packages/coding-agent/src/extensions/codemode/execute.ts:50` | Cost / rate-limit protection | — |
| codemode store limits · [[code-mode]] | `MAX_STORE_VALUE_CHARS` / `MAX_STORE_TOTAL_CHARS` = 256 KiB / 1 MiB | `packages/codemode/src/runtime/prelude-source.ts:32-33` | — | — |
| codemode raw output · [[code-mode]] | `MAX_OUTPUT_CHARS` / `MAX_OUTPUT_ITEMS` = 16 MiB / 100_000 | `packages/codemode/src/runtime/prelude-source.ts:40-41` | Prevent host OOM from print loops | — |
| codemode preview chars · [[code-mode]] | `ARGS_PREVIEW_CHARS` / `ERROR_PREVIEW_CHARS` = 200 / 500 | `packages/coding-agent/src/extensions/codemode/execute.ts:47-48` | — | — |
| Clipboard OSC52 cap | `MAX_OSC52_ENCODED_LENGTH` = `100_000` | `packages/coding-agent/src/utils/clipboard.ts:9` | Terminal limits | — |
| Clipboard command timeout/buffer | 3000 ms / 50 MiB | `packages/coding-agent/src/utils/clipboard-command.ts:30,38` | — | — |
| Durable truncation (mirror) · [[tool-output-truncation]] | `DEFAULT_MAX_LINES/BYTES` = 2000 / 50KB | `packages/durable/src/truncate.ts:11-12` | Copied into new harness | a5b27367d |
| Durable read chunk · [[file-read-tool]] | `READ_CHUNK` = `64 KiB` | `packages/durable/src/tools/read.ts:39` | — | — |
| Durable bash max timeout · [[shell-execution]] · [[no-bash-default-timeout]] | `MAX_TIMEOUT_SECONDS` = `2_147_483_647/1000` | `packages/durable/src/tools/bash.ts:8` | — | — |
| Subagent example limits · [[subagent-as-subprocess]] · [[no-subagents-core]] | `MAX_PARALLEL_TASKS` / `MAX_CONCURRENCY` / `PER_TASK_OUTPUT_CAP` = 8 / 4 / 50 KiB | `packages/coding-agent/examples/extensions/subagent/index.ts:33-36` | Example only (subagents not core) | — |

## 04-prompting

| constant (neutral) | pi | pi location | tradeoff | history |
|---|---|---|---|---|
| Skill name max · [[skill-progressive-disclosure]] | `MAX_NAME_LENGTH` = `64` | `packages/coding-agent/src/core/skills.ts:11` | Agent Skills spec | 05b7b8133 (2025-12-19 "Skills standard compliance") |
| Skill description max · [[skill-progressive-disclosure]] | `MAX_DESCRIPTION_LENGTH` = `1024` | `packages/coding-agent/src/core/skills.ts:14` | Spec; bounds system-prompt skill list | 05b7b8133 |
| MCP servers system-prompt section · [[mcp-integration]] · [[cache-stable-prompt-prefix]] | `MAX_SERVERS_SECTION_CHARS` = `4096` | `packages/coding-agent/src/extensions/mcp/index.ts:163` | Deferred servers listed cheaply | e029c3ed0 (2026-09-30) |
| MCP server description chars · [[mcp-integration]] · [[cache-stable-prompt-prefix]] | `MAX_SERVER_DESCRIPTION_CHARS` = `250` | `packages/coding-agent/src/extensions/mcp/index.ts:158` | "as Codex allows for deferred namespaces" | e029c3ed0 |
| codemode inline declaration budget · [[code-mode]] | `DEFAULT_CODEMODE_INLINE_BUDGET` / `codemode.inlineBudget` = `3000` tokens (chars/4) | `packages/coding-agent/src/extensions/codemode/tool.ts:156,158` | Overflow tools found via `searchTools()` | 8562bcf66 |
| codemode input schema size · [[code-mode]] | `DEFAULT_INPUT_SCHEMA_MAX_CHARS` = `16_000` | `packages/codemode/src/declarations.ts:10` | — | — |
| codemode $ref expansions · [[code-mode]] | `MAX_REF_EXPANSIONS` = `32` | `packages/codemode/src/declarations.ts:12` | Recursive schema guard | — |
| Mid-run MCP wait at first prompt · [[mcp-integration]] | `DEFAULT_STARTUP_WAIT_MS` = `10_000` | `packages/coding-agent/src/extensions/mcp/index.ts:93` | Only servers with direct tools block first prompt | e029c3ed0 |
| Widget lines · [[extension-ui-primitives]] | `MAX_WIDGET_LINES` = `10` | `packages/coding-agent/src/modes/interactive/interactive-mode.ts:2477` | — | — |

## 05-context

| constant (neutral) | pi | pi location | tradeoff | history |
|---|---|---|---|---|
| Compaction enabled · [[auto-compaction]] | `compaction.enabled` = `true` | `packages/coding-agent/src/core/compaction/compaction.ts:127` | — | 6c2360af2 (2025-12-04, #92) |
| Compaction reserve · [[auto-compaction]] | `reserveTokens` = `16384` | `packages/coding-agent/src/core/compaction/compaction.ts:128`; `packages/coding-agent/src/core/settings-manager.ts:24` | Trigger at `ctx > window - 16384`; "~8k summary + ~8k safety" | 5daef11b4 plan (2025-12-02); never changed; per-model overrides 46bde88a1 (2026-09-07, #8133) |
| Compaction keep-recent · [[auto-compaction]] | `keepRecentTokens` = `20000` | `packages/coding-agent/src/core/compaction/compaction.ts:129`; `packages/coding-agent/src/core/settings-manager.ts:25` | Verbatim tail; "~20k" borrowed from Codex compact.rs | 5daef11b4 cites `codex-rs/core/src/compact.rs`; never changed |
| Compaction trigger · [[auto-compaction]] | `shouldCompact` = `ctx > window - reserve` | `packages/coding-agent/src/core/compaction/compaction.ts:267-270` | No % threshold; fixed headroom | — |
| Summary output budget · [[auto-compaction]] | `floor(0.8 * reserveTokens)` (=13107) | `packages/coding-agent/src/core/compaction/compaction.ts:713` | Leaves 20% of reserve for prompt overhead | 6c2360af2 |
| Turn-prefix summary budget · [[split-turn-summary]] | `floor(0.5 * reserveTokens)` (=8192) | `packages/coding-agent/src/core/compaction/compaction.ts:1091` | Split-turn prefix gets smaller budget | a38e61909 (2025-12-09) |
| Tool-result chars in summarizer input · [[transcript-serialization-for-summary]] | `TOOL_RESULT_MAX_CHARS` = `2000` | `packages/coding-agent/src/core/compaction/utils.ts:94` | Prevent summarization request itself overflowing | c950c692a (2026-03-06, #1796) |
| Compaction token heuristic · [[token-estimation]] | `estimateTokens` = `chars / 4` | `packages/coding-agent/src/core/compaction/compaction.ts:298-344` | "Conservative (overestimates)" — comment | since fd13b53b1/earlier |
| Request token heuristic (ai) · [[token-estimation]] | `CHARS_PER_TOKEN` = `3.5` | `packages/ai/src/utils/estimate.ts:15` | More tokens per char → safer max_tokens clamp | 27075fe07 (2026-10-06, #10497): **4 → 3.5** (DeepSeek V4 Flash overflow) |
| Image token estimate · [[token-estimation]] · [[image-normalization]] | `ESTIMATED_IMAGE_CHARS` = `4800` (≈1200 tok @4, ≈1371 @3.5) | `packages/ai/src/utils/estimate.ts:16`; `packages/coding-agent/src/core/compaction/compaction.ts:276` | Duplicated in two estimators | — |
| Branch summary reserve · [[branch-summary]] | `branchSummary.reserveTokens` = `16384` | `packages/coding-agent/src/core/settings-manager.ts:1001`; `packages/coding-agent/src/core/compaction/branch-summarization.ts:305` | — | dc5fc4fc4 (2025-12-29): replaced `maxTokens = 100000` with reserve 16384 |
| Branch summary fallback window · [[branch-summary]] | `128000` | `packages/coding-agent/src/core/compaction/branch-summarization.ts:312` | — | — |
| Durable compaction policy · [[background-compaction]] · [[auto-compaction]] | `DEFAULT_COMPACTION_POLICY` = 16384 / 20000 / background 32768 | `packages/durable/src/harness/agent.ts:26-31` | New: background compaction starts 32k below blocking threshold | ed0d6b91b (2026-09-30), b56702ad3 |
| Durable context retention · [[durable-execution]] | `DEFAULT_CONTEXT_RETENTION_MS` = `600_000` (10 min) | `packages/durable/src/harness/agent.ts:53` | (unverified semantics: in-memory context lifetime) | — |
| Durable summarizer tool-result cap · [[transcript-serialization-for-summary]] | `TOOL_RESULT_MAX_CHARS` = `2000` | `packages/durable/src/harness/compaction.ts:55` | Mirror | ed0d6b91b |
| Images auto-resize · [[image-normalization]] | `images.autoResize` = `true` | `packages/coding-agent/src/core/settings-manager.ts:1427` | — | — |
| Block images · [[image-normalization]] | `images.blockImages` = `false` | `packages/coding-agent/src/core/settings-manager.ts:1440` | — | — |
| Context % display · [[token-estimation]] | `tokens/contextWindow*100` | `packages/coding-agent/src/core/agent-session.ts:4272` | Display only | — |
| Transcript script char cap · [[token-estimation]] | `MAX_CHARS_PER_FILE` = `100_000` ("~20k tokens") | `scripts/session-transcripts.ts:20` | Note: comment implies 5 chars/token | — |

## 06-caching

| constant (neutral) | pi | pi location | tradeoff | history |
|---|---|---|---|---|
| Anthropic cache lifetimes · [[cache-retention-control]] | `ANTHROPIC_PROMPT_CACHE` = `{short: 300, long: 3600}` s | `packages/ai/scripts/generate-models.ts:983` | Only direct Anthropic annotated; OpenAI deliberately not | c596d09d9 (2026-09-19, #9668) |
| Long retention TTL · [[cache-retention-control]] | `ttl: "1h"` when `retention === "long"` | `packages/ai/src/api/anthropic-messages.ts:88` | Higher write cost vs survival | — |
| Default retention · [[cache-retention-control]] | `PI_CACHE_RETENTION` = `"short"` unless env `long` | `packages/coding-agent/src/core/cache-warmer.ts:42` | — | — |
| Cache warming mode · [[cache-warming]] | `cacheWarming` = `"streaming"` (global only) | `packages/coding-agent/src/core/settings-manager.ts:1048` (doc `:182`) | Keep cache warm during runs; "each refresh costs money" | c596d09d9 |
| Warm refresh point · [[cache-warming]] | `getCacheWarmingDelayMs` = `min(0.9·TTL, TTL−10s)` (5m → 270s) | `packages/coding-agent/src/core/cache-warmer.ts:29-31` | — | c596d09d9 |
| Streaming warming horizon · [[cache-warming]] | `MAX_WARMING_AGE_MS` = `60 min` | `packages/coding-agent/src/core/cache-warmer.ts:16` | — | c596d09d9 |
| Idle warming horizon · [[cache-warming]] | `MAX_IDLE_WARMING_AGE_MS` = `30 min` | `packages/coding-agent/src/core/cache-warmer.ts:18` | Continuation estimates degrade | c596d09d9 |
| Min expected savings · [[cache-warming]] | `CACHE_WARMING_MINIMUM_EXPECTED_SAVINGS` = `$0.05` | `packages/coding-agent/src/core/cache-warmer.ts:20` | EV-gated refresh | c596d09d9 |
| Idle continuation probability · [[cache-warming]] | `IDLE_CONTINUATION_PROBABILITY` = `0.15` | `packages/coding-agent/src/core/cache-warmer.ts:26` | "Measured from our own usage; per-session estimates were not better" | c596d09d9 |
| Warm probe output · [[cache-warming]] | `maxTokens: 1` | `packages/coding-agent/src/core/cache-warmer.ts:335` | Cheapest refresh; not for budget-thinking Claude (`isReplayable`) | c596d09d9 |
| Cache miss idle threshold · [[cache-miss-accounting]] | `CACHE_TTL_MS` = `5 min` | `packages/coding-agent/src/core/cache-stats.ts:8` | Attribute misses to idle gap | 3f9aa5d10 (2026-07-09, #6427) |
| Cache miss noise floor · [[cache-miss-accounting]] | `NOISE_FLOOR_TOKENS` = `1024` | `packages/coding-agent/src/core/cache-stats.ts:11` | Breakpoint granularity noise | 3f9aa5d10 |
| Summaries skip cache writes · [[cache-retention-control]] | `cacheRetention: "none"` | `packages/coding-agent/src/core/compaction/compaction.ts:631` | One-off request shouldn't pay write premium | — |
| Show cache-miss notices · [[cache-miss-accounting]] | `showCacheMissNotices` = `false` | `packages/coding-agent/src/core/settings-manager.ts:1074` | — | — |

## 07-safety

| constant (neutral) | pi | pi location | tradeoff | history |
|---|---|---|---|---|
| Project trust default · [[project-trust-gate]] | `defaultProjectTrust` = `"ask"` (global only) | `packages/coding-agent/src/core/settings-manager.ts:151` | Project extensions/settings gated | — |
| Output file mode · [[tool-output-spill]] | `OUTPUT_FILE_MODE` = `0o600` | `packages/coding-agent/src/utils/output-files.ts:15` | Temp full-output files private | — |
| Unix socket mode · [[client-server-session-split]] | `DEFAULT_SOCKET_MODE` = `0o600` | `packages/server/src/transports/unix/listener.ts:11` | — | — |
| MCP OAuth refresh skew / timeout · [[mcp-integration]] · [[subscription-oauth-auth]] | `REFRESH_SKEW_MS` / `REFRESH_REQUEST_TIMEOUT_MS` = 30s / 15s | `packages/coding-agent/src/extensions/mcp/oauth.ts:46,48` | — | — |
| MCP OAuth refresh lock · [[mcp-integration]] · [[subscription-oauth-auth]] | `REFRESH_LOCK_STALE_MS` / `WAIT` / `RETRY` = 20s / 25s / 100ms | `packages/coding-agent/src/extensions/mcp/oauth.ts:50-53` | Cross-process refresh races | — |
| MCP login timeout · [[mcp-integration]] | `DEFAULT_LOGIN_TIMEOUT_SECONDS` = `300` | `packages/coding-agent/src/extensions/mcp/cli.ts:77` | — | — |
| Auth file lock (sync) · [[subscription-oauth-auth]] · [[credential-resolution]] | 10 attempts × 20ms busy-wait | `packages/coding-agent/src/core/auth-storage.ts:70-71` | Sync API kept | — |
| Auth file lock (async) · [[subscription-oauth-auth]] · [[credential-resolution]] | stale 30s, max delay 2s, jittered | `packages/coding-agent/src/core/auth-storage.ts:120-121,144` | — | — |
| Settings lock · [[layered-settings]] | 10 × 20ms | `packages/coding-agent/src/core/settings-manager.ts:328-329` | — | — |
| Trust file lock · [[project-trust-gate]] | 10 × 20ms | `packages/coding-agent/src/core/trust-manager.ts:141-142` | — | — |
| Crash log retention | `MAX_CRASH_RECORDS` / `MAX_AGE` = 5 / 7 days | `packages/coding-agent/src/core/crash-log.ts:18-19` | — | — |
| Install telemetry · [[install-telemetry]] | `enableInstallTelemetry` = `true` | `packages/coding-agent/src/core/settings-manager.ts:1165` | Opt-out | — |
| Analytics · [[install-telemetry]] | `enableAnalytics` = `false` | `packages/coding-agent/src/core/settings-manager.ts:1175` | Opt-in | — |
| Anthropic extra-usage warning · [[usage-cost-accounting]] | `warnings.anthropicExtraUsage` = `true` | `packages/coding-agent/src/core/settings-manager.ts:91` | — | — |
| Lockfile check · [[supply-chain-pinning]] | `PI_ALLOW_LOCKFILE_CHANGE=1` = env override | `scripts/check-lockfile-commit.mjs:119` | Repo hygiene | — |
| Model catalog publish floor · [[model-catalog]] | `MINIMUM_MODEL_COUNT` = `500` | `scripts/publish-model-catalog.mjs:33` | Guard against publishing truncated catalog | — |

## 08-state

| constant (neutral) | pi | pi location | tradeoff | history |
|---|---|---|---|---|
| Session format version · [[session-migration]] | `CURRENT_SESSION_VERSION` = `3` | `packages/coding-agent/src/core/session-manager.ts:41` | Migrations on load | — |
| Session read buffer · [[session-tree]] | `SESSION_READ_BUFFER_SIZE` = `1 MiB` | `packages/coding-agent/src/core/session-manager.ts:604` | — | — |
| Session header read buffer · [[session-tree]] | `SESSION_HEADER_READ_BUFFER_SIZE` = `4096` | `packages/coding-agent/src/core/session-manager.ts:605` | — | — |
| Header scan bound · [[session-tree]] | `MAX_SESSION_HEADER_SCAN_BYTES` = `1 MiB` | `packages/coding-agent/src/core/session-manager.ts:607` | Large cwd/metadata allowed | — |
| Session list concurrency · [[session-tree]] | `MAX_CONCURRENT_SESSION_INFO_LOADS` / `DISCOVERY_LOADS` = 10 / 64 | `packages/coding-agent/src/core/session-manager.ts:891-892` | — | — |
| Session list publish batch · [[session-tree]] | `CURRENT_/ALL_SESSION_LIST_PUBLISH_INTERVAL` = 10 / 100 | `packages/coding-agent/src/core/session-manager.ts:893-894` | Progressive UI | — |
| Session file size limit · [[session-tree]] | **none** (unverified: no cap found) | — | Append-only JSONL grows unbounded | absence |
| Bug report schema | `BUG_REPORT_SCHEMA_VERSION` = `1` | `packages/coding-agent/src/core/bug-report.ts:18` | — | — |
| Footer git watch debounce · [[extension-ui-primitives]] | `WATCH_DEBOUNCE_MS` = `500` | `packages/coding-agent/src/core/footer-data-provider.ts:101` | — | — |
| FS watch retry | `FS_WATCH_RETRY_DELAY_MS` = `5000` | `packages/coding-agent/src/utils/fs-watch.ts:3` | — | — |
| MCP log size · [[mcp-integration]] | `MAX_LOG_BYTES` = `5 MiB` | `packages/coding-agent/src/extensions/mcp/log.ts:10` | — | — |
| SQLite WAL / busy · [[durable-execution]] | `DEFAULT_WAL_AUTO_CHECKPOINT_PAGES` / `DEFAULT_BUSY_TIMEOUT_MS` = 1000 / 5000 | `packages/durable/src/storage/sqlite/node.ts:16-17` | — | — |
| Durable scan page · [[durable-execution]] | `SCAN_PAGE_SIZE` = `256` | `packages/durable/src/harness/harness.ts:57` (+5 sites) | — | — |
| Durable JSONL format · [[durable-execution]] | `FORMAT_VERSION` = `1` | `packages/durable/src/storage/jsonl/storage.ts:28` | — | 898ab8040 |
| Durable pending watch frames · [[durable-execution]] | `MAX_PENDING_WATCH_FRAMES` = `100` | `packages/durable/src/session/observation.ts:15` | — | — |
| Durable output progress rate · [[durable-execution]] | `PROGRESS_BYTES_PER_SECOND` = `100 KiB/s` | `packages/durable/src/harness/output.ts:261` | — | — |
| Durable watcher · [[remote-execution-env]] | `DEBOUNCE_MS`/`FSEVENTS_SETTLE_MS`/`DEFAULT_POLL_MS`/`DEFAULT_MAX_DIRECTORIES`/`HASH_MAX_BYTES` = 50 / 500 / 2000 / 10_000 / 256 KiB | `packages/durable/src/env/node-watch.ts:46-58` | — | — |
| Durable spill high-water · [[remote-execution-env]] | `SPILL_HIGH_WATER_MARK` = `1 MiB` | `packages/durable/src/env/node.ts:53` | — | — |
| Chord delta limits · [[replicated-state]] | `MAX_DELTA_OPERATIONS` / `MAX_IDENTITY_CANDIDATES` / `MAX_SEMANTIC_CELLS` / `DEFAULT_OVERLAP_SCAN` = 4096 / 200_000 / 65_536 / 65_536 | `packages/chord/src/delta/diff.ts:4-5,86-87` | Bounded diff cost | — |

## 09-subagents
Scope: subagents and multi-process (no core subagent tool — see [[no-subagents-core]]).

Subagents exist only as an example extension (`subagent/`) and the experimental session-worker/coordinator.

| constant (neutral) | pi | pi location | tradeoff | history |
|---|---|---|---|---|
| Example subagent parallelism · [[subagent-as-subprocess]] · [[no-subagents-core]] | `MAX_PARALLEL_TASKS` / `MAX_CONCURRENCY` = 8 / 4 | `packages/coding-agent/examples/extensions/subagent/index.ts:33-34` | — | — |
| Example per-task output cap · [[subagent-as-subprocess]] · [[no-subagents-core]] | `PER_TASK_OUTPUT_CAP` = `50 KiB` | `packages/coding-agent/examples/extensions/subagent/index.ts:36` | = tool truncation limit | — |
| Worker startup/shutdown/discovery/demand · [[client-server-session-split]] | `WORKER_*_TIMEOUT_MS` = 15s / 10s / 5s / 5s | `packages/coding-agent/src/experimental/session-worker-manager.ts:31-34` | — | — |
| Worker demand grace · [[client-server-session-split]] | `DEFAULT_INITIAL/ORPHAN_DEMAND_GRACE_MS` = 10s / 30s | `packages/coding-agent/src/experimental/session-worker.ts:307-308` | — | — |
| Worker lock retries · [[client-server-session-split]] | 320 × 25ms, max 8s | `packages/coding-agent/src/experimental/session-worker.ts:521` | — | — |
| Coordinator · [[client-server-session-split]] | `COORDINATOR_PROTOCOL_VERSION` / start timeout / retry = 3 / 10s / 10ms | `packages/coding-agent/src/experimental/coordinator.ts:14-16` | — | — |
| Coordinator empty grace · [[client-server-session-split]] | `EMPTY_STARTUP/SHUTDOWN_GRACE_MS` = 30s / 250ms | `packages/coding-agent/src/experimental/coordinator.ts:263-264` | — | — |
| Server lock · [[client-server-session-split]] | `LOCK_STALE_MS` / `LOCK_RETRY_MS` / `LOCK_WAIT_MS` = 30s / 25ms / 30s | `packages/coding-agent/src/experimental/server.ts:73-75` | — | — |
| Auto-server grace · [[client-server-session-split]] | `AUTO_SERVER_STARTUP/IDLE_GRACE_MS` = 10s / 1s | `packages/coding-agent/src/experimental/server.ts:244-245` | — | — |
| Control line max · [[client-server-session-split]] | `MAX_CONTROL_LINE_BYTES` = `128 MiB` | `packages/coding-agent/src/experimental/process.ts:100` | — | — |
| Radius relay backoff · [[client-server-session-split]] | `HOST/CLIENT_RETRY_INITIAL_MS` / `_MAX_MS` = 1s → 30s | `packages/coding-agent/src/experimental/radius-relay.ts:16-20` | — | — |
| Relay drain threshold · [[client-server-session-split]] | `DRAIN_THRESHOLD_BYTES` = `1 MiB` | `packages/coding-agent/src/experimental/radius-relay.ts:15` | — | — |

## 10-platform
Scope: TUI, protocol, remote env, MCP transport, packaging.

| constant (neutral) | pi | pi location | tradeoff | history |
|---|---|---|---|---|
| TUI frame budget · [[differential-tui-rendering]] | `MIN_RENDER_INTERVAL_MS` = `16` ms (~60fps) | `packages/tui/src/tui.ts:507` | Coalesce `requestRender()` under streaming; `force` bypasses | 6f5f37f85 (2026-04-06) |
| Max render write · [[differential-tui-rendering]] | `MAX_RENDER_WRITE_CHARS` = `1 MiB` | `packages/tui/src/tui-main-screen.ts:9` | — | — |
| Lone-ESC timeout · [[differential-tui-rendering]] | `DEFAULT_ESCAPE_TIMEOUT_MS` = `10` ms (100 over SSH) | `packages/tui/src/terminal.ts:115-116` | Esc responsiveness vs split Alt+key sequences | 06ed87167 (2026-08-11, #7899), 2a95ef70d |
| Stdin sequence timeouts · [[differential-tui-rendering]] | `DEFAULT_SEQUENCE_TIMEOUT_MS` / `DEFAULT_ESCAPE_TIMEOUT_MS` = 50 / 10 | `packages/tui/src/stdin-buffer.ts:23-24` | — | — |
| Kitty keyboard flags / reply timeout · [[differential-tui-rendering]] | `DESIRED_KITTY_KEYBOARD_PROTOCOL_FLAGS` / `..._FRAGMENT_TIMEOUT_MS` = 7 / 150ms | `packages/tui/src/terminal.ts:12-13` | — | — |
| Terminal progress keepalive · [[differential-tui-rendering]] | `TERMINAL_PROGRESS_KEEPALIVE_MS` = `1000` | `packages/tui/src/terminal.ts:8` | — | — |
| Theme query timeout · [[differential-tui-rendering]] | `TERMINAL_QUERY_TIMEOUT_MS` = `100` | `packages/coding-agent/src/modes/interactive/theme/theme-controller.ts:25` | — | — |
| Loader spinner · [[differential-tui-rendering]] | `DEFAULT_INTERVAL_MS` = `80` | `packages/tui/src/components/loader.ts:12` | — | — |
| Wheel scroll · [[differential-tui-rendering]] | `BURST_GAP_MS`/`GESTURE_GAP_MS`/`REFERENCE_GAP_MS`/`MAX_AUTO_LINES` = 5 / 200 / 100 / 6 | `packages/tui/src/wheel-scroll.ts:6-11` | — | — |
| Alt-screen · [[differential-tui-rendering]] | `PAGE_SCROLL_OVERLAP`/`ALT_WHEEL_SCROLL_MULTIPLIER`/`DOUBLE_CLICK_INTERVAL_MS` = 4 / 5 / 500 | `packages/tui/src/tui-alt-screen.ts:76-81` | — | — |
| Offscreen kitty image cache · [[differential-tui-rendering]] | `MAX_CACHED_OFFSCREEN_KITTY_*` = 16 imgs / 32 MiB tx / 64 MiB decoded | `packages/tui/src/tui-alt-screen.ts:78-80` | — | — |
| Width cache · [[differential-tui-rendering]] | `WIDTH_CACHE_SIZE` = `512` | `packages/tui/src/utils.ts:51` | — | — |
| Inline image width · [[image-normalization]] | `terminal.imageWidthCells` = `60` | `packages/coding-agent/src/core/settings-manager.ts:59` | — | — |
| Autocomplete visible · [[differential-tui-rendering]] | `autocompleteMaxVisible` = `5` (3–20) | `packages/coding-agent/src/core/settings-manager.ts:1522` | — | — |
| TUI mode · [[differential-tui-rendering]] | `tuiMode` = `"fullscreen"` | `packages/coding-agent/src/core/settings-manager.ts:1372` | — | — |
| Protocol version · [[client-server-session-split]] | `PROTOCOL_VERSION` = `8` | `packages/protocol/src/protocol.ts:5` | — | — |
| Frame size · [[client-server-session-split]] | `DEFAULT_MAX_FRAME_LENGTH` = `16 MiB` (4-byte header) | `packages/protocol/src/framing.ts:1,6` | — | — |
| CBOR limits · [[client-server-session-split]] | `DEFAULT_MAX_CBOR_BYTE_LENGTH`/`CONTAINER_LENGTH`/`DEPTH` = 16 MiB / 1_000_000 / 64 | `packages/protocol/src/cbor/options.ts:6-8` | DoS bounds | — |
| Server handshake · [[client-server-session-split]] | `DEFAULT_HANDSHAKE_TIMEOUT_MS` = `5_000` | `packages/server/src/server.ts:42` | — | — |
| Unix listener close / probe · [[client-server-session-split]] | `DEFAULT_GRACEFUL_CLOSE_TIMEOUT_MS` / `SOCKET_PROBE_TIMEOUT_MS` = 5s / 1s | `packages/server/src/transports/unix/listener.ts:12,15` | — | — |
| Client discovery · [[client-server-session-split]] | `DEFAULT_DISCOVERY_TIMEOUT_MS` / `MAX_CONCURRENT_DISCOVERY_PROBES` = 1s / 16 | `packages/client/src/unix.ts:14,17` | — | — |
| Env (remote) frame / heartbeat · [[remote-execution-env]] · [[remote-host-trust]] | `MAX_FRAME` / `PING_INTERVAL_MS` / `SILENCE_LIMIT_MS` / `START_TIMEOUT_MS` = 16 MiB / 5s / 30s / 60s | `packages/env/src/connection.ts:12-17` | 6 missed pings = dead | ba03e03f2 (2026-10-05) |
| Env remote IO chunks · [[remote-execution-env]] · [[remote-host-trust]] | `READ_CHUNK`/`WRITE_CHUNK`/`READ_DEPTH` = 256 KiB / 512 KiB / 8 | `packages/env/src/remote-env.ts:36-40` | Pipelined IO | ba03e03f2 |
| SSH timeouts · [[remote-execution-env]] · [[remote-host-trust]] | `SSH_TIMEOUT_MS` / `UPLOAD_TIMEOUT_MS` / `ServerAliveInterval` = 60s / 300s / 15 | `packages/env/src/ssh.ts:123-124,103` | — | ba03e03f2 |
| Env watch reconnect · [[remote-execution-env]] · [[remote-host-trust]] | `RECONNECT_FIRST_MS` / `RECONNECT_MAX_MS` = 1s / 30s | `packages/env/src/watch.ts:16-17` | — | — |
| MCP request timeout (client lib) · [[mcp-integration]] | `DEFAULT_REQUEST_TIMEOUT_MS` = `30_000` | `packages/mcp/src/client.ts:38` | (coding-agent overrides with 60s per server) | 8562bcf66 |
| MCP list pagination · [[mcp-integration]] | `MAX_LIST_PAGES` = `1_000` | `packages/mcp/src/client.ts:39` | Infinite cursor guard | — |
| MCP max message · [[mcp-integration]] | `DEFAULT_MAX_MESSAGE_BYTES` = `16 MiB` | `packages/mcp/src/transports/transport.ts:3` | — | — |
| MCP stdio stderr / close · [[mcp-integration]] | `DEFAULT_MAX_STDERR_BYTES` / `DEFAULT_CLOSE_TIMEOUT_MS` / `STDIN_CLOSE_GRACE_MS` = 64 KiB / 2s / 500ms | `packages/mcp/src/transports/stdio.ts:7-10` | — | 8562bcf66 |
| MCP streamable-http reconnect · [[mcp-integration]] | `DEFAULT_RECONNECT_INITIAL/MAX_DELAY_MS` / `MAX_RETRIES` = 1s / 30s / 5 | `packages/mcp/src/transports/streamable-http.ts:16-18` | — | — |
| MCP HTTP error body · [[mcp-integration]] | `MAX_ERROR_BODY_BYTES` / `ERROR_MESSAGE_BODY_CHARS` = 8 KiB / 500 | `packages/mcp/src/transports/streamable-http.ts:14-15` | — | — |
| Package manager network · [[harness-package-distribution]] | `NETWORK_TIMEOUT_MS` / concurrency = 10s / 4 / 4 | `packages/coding-agent/src/core/package-manager.ts:50-52` | — | — |
| Tool binary download · [[search-tools]] | `NETWORK_TIMEOUT_MS` / `DOWNLOAD_TIMEOUT_MS` = 10s / 120s | `packages/coding-agent/src/utils/tools-manager.ts:11-12` | fd/rg auto-download | — |
| Version check | `DEFAULT_VERSION_CHECK_TIMEOUT_MS` = `10000` | `packages/coding-agent/src/utils/version-check.ts:6` | — | — |
| Config value command · [[credential-resolution]] | `timeout: 10000` | `packages/coding-agent/src/core/resolve-config-value.ts:160,189` | `!cmd` API-key resolution | — |
| Shell detection · [[shell-execution]] | `timeout: 5000` | `packages/coding-agent/src/utils/shell.ts:30,47` | — | — |
| Clipboard image · [[image-normalization]] | `DEFAULT_LIST_TIMEOUT_MS` / `DEFAULT_POWERSHELL_TIMEOUT_MS` = 1s / 5s | `packages/coding-agent/src/utils/clipboard-image.ts:19-20` | — | — |
| Release announcement | `RETRY_DELAY_MS`/`RETRY_TIMEOUT_MS`/`MAX_POINTER_UPDATE_ATTEMPTS` = 5s / 10 min / 5 | `scripts/publish-release-announcement.mjs:14-16` | — | — |

## Top 25

1. **No max-turn / step cap** ([[no-turn-cap]], [[turn-loop]]) — `packages/agent/src/*` has no numeric literals. The only bounds are context ([[auto-compaction]]), retries ([[auto-retry-backoff]]) and the user's Esc ([[abort-propagation]]). `packages/agent/README.md:146` warns that an unconditional `finishTurn` "continue" creates an endless loop.
2. **bash has no default timeout** ([[no-bash-default-timeout]], [[shell-execution]]) — `packages/coding-agent/src/core/tools/bash.ts:42`. It was 30s until `29900ce64` (2025-11-12), which changed it to "commands run until completion unless specified". The model opts in with `timeout` (seconds), capped at int32 ms (`packages/coding-agent/src/core/tools/bash.ts:22`; validated `cbcf4e04c`, `85b7c2474`).
3. **Tool output limit: 2000 lines / 50KB, whichever hits first** ([[tool-output-truncation]]) — `packages/coding-agent/src/core/tools/truncate.ts:11-12`. Introduced at 30KB in `de77cd141` and raised to 50KB hours later in `306f9cc66` (#134). `read` cuts the head and points to `offset`; `bash` keeps the tail and spills the full output to a temp file ([[tool-output-spill]]). The same values are copied into `packages/durable/src/truncate.ts:11-12`.
4. **grep/find/ls limits 100/1000/500, grep line cap 500 chars** ([[search-tools]]) — `de77cd141`, `b813a8b92`. The line cap stops minified files from consuming the byte budget.
5. **`compaction.reserveTokens = 16384`** ([[auto-compaction]]) — `packages/coding-agent/src/core/compaction/compaction.ts:128`. The trigger is `ctx > window − 16384` (`:267-270`): fixed headroom, not a percentage. The plan doc `5daef11b4` splits it as roughly 8k for the summary and 8k of safety. Never changed; per-model overrides came in `46bde88a1` (#8133).
6. **`keepRecentTokens = 20000`** ([[auto-compaction]], [[compaction-cut-point]]) — `packages/coding-agent/src/core/compaction/compaction.ts:129`. Borrowed from Codex `compact.rs` ("last ~20k tokens") per `5daef11b4`.
7. **Summary budgets: 0.8 × reserve (13107) for the main summary, 0.5 × reserve (8192) for a split-turn prefix** ([[split-turn-summary]]) — `packages/coding-agent/src/core/compaction/compaction.ts:713,1091`.
8. **Summarizer input caps each tool result at 2000 chars** ([[transcript-serialization-for-summary]]) — `packages/coding-agent/src/core/compaction/utils.ts:94`. Added in `c950c692a` (#1796) after the compaction request itself overflowed.
9. **Two chars-per-token heuristics disagree** ([[token-estimation]]) — compaction uses `chars/4` (`packages/coding-agent/src/core/compaction/compaction.ts:298-344`, commented "conservative"). The ai max_tokens clamp uses `3.5` (`packages/ai/src/utils/estimate.ts:15`), lowered from 4 in `27075fe07` (2026-10-06, #10497) after a DeepSeek V4 Flash overflow. The "conservative" compaction estimate now *under*-counts relative to the clamp.
10. **`CONTEXT_SAFETY_TOKENS = 4096`** ([[max-tokens-context-clamp]]) — `packages/ai/src/api/simple-options.ts:15`. Sets `maxTokens = min(req, window − estimate − 4096)` (`09f105957`).
11. **Agent retry: 3 attempts at 2s/4s/8s, each wait capped at 60s** ([[auto-retry-backoff]]) — `packages/coding-agent/src/core/settings-manager.ts:1026-1027`, `packages/ai/src/utils/retry.ts:126`. Came with `bb445d24f` (#157), which also set SDK retries to 0. `8fb1e877c` later removed "hidden 429 retries", so every retry is visible and abortable.
12. **Provider-level backoff `min(0.5·2^n, 8)s` with 25% jitter** — `packages/ai/src/utils/provider-retry.ts:67-68`. Mirrors the SDK policy but only runs when the user opts in (`maxRetries ?? 0`, `:111`).
13. **`maxRetryDelayMs = 60s`** — `packages/ai/src/utils/provider-retry.ts:1`. A longer server `retry-after` fails fast instead of being honored. Added in `030a61d88` because Gemini CLI asked for waits of hours.
14. **HTTP idle timeout 300s; happy-eyeballs attempt timeout 2s** ([[http-transport-hardening]]) — `packages/coding-agent/src/core/http-dispatcher.ts:4,6`. 300s covers long thinking pauses (`849f9d9c5`, #4759); Node's 250ms happy-eyeballs default was too short for high-latency connects.
15. **Anthropic cache lifetimes `{short: 300, long: 3600}` s** ([[cache-retention-control]]) — `packages/ai/scripts/generate-models.ts:983`. Annotated only for direct Anthropic; OpenAI TTLs are deliberately not annotated (`c596d09d9`).
16. **Cache warming is expected-value gated** ([[cache-warming]]) — refresh at `min(0.9·TTL, TTL−10s)` with a 1-token probe (`packages/coding-agent/src/core/cache-warmer.ts:29-31,335`). It runs only when expected savings ≥ $0.05, using an idle continuation probability of 0.15 "measured from our own usage" (`:20,26`), with horizons of 60 min while streaming and 30 min idle (`:16,18`). A rare explicit cost model in harness code (`c596d09d9`).
17. **Cache-miss accounting: 5-min TTL heuristic, 1024-token noise floor** ([[cache-miss-accounting]]) — `packages/coding-agent/src/core/cache-stats.ts:8,11`, `3f9aa5d10` (#6427).
18. **Images: 2000×2000 px, 4.5 MiB base64, JPEG quality 80 then 85/70/55/40** ([[image-normalization]]) — `packages/coding-agent/src/utils/image-resize-core.ts:32-38,132`. The 4.5 MiB cap leaves headroom under Anthropic's 5MB (`69dc6b078`, #424). Generalized to per-provider `inputLimits` in `f5c946480` (#9631).
19. **Thinking budgets 1024/2048/8192/16384, `MIN_ANSWER_TOKENS` 1024** ([[thinking-level-abstraction]]) — `packages/ai/src/api/simple-options.ts:68-75`. xhigh/max clamp to 16384 on budget-based Claude.
20. **Fallback metadata for unknown/custom models: 128000 context / 16384 output** ([[model-catalog]]) — `packages/coding-agent/src/core/provider-composer.ts:242-243`. The generator falls back to 4096 when models.dev has no data (`packages/ai/scripts/generate-models.ts:1494-1495`).
21. **MCP limits** ([[mcp-integration]], [[no-builtin-mcp-reversed]]) — 60s per-call timeout; 20 KiB output cap with a middle cut (smaller than the built-in 50KB); a 10s first-prompt wait that applies only to direct-tool servers; a 4096-char servers section with 250-char descriptions "as Codex allows" (`8562bcf66`, `e029c3ed0`).
22. **Codemode sandbox** ([[code-mode]]) — 256 MiB QuickJS heap, 300s library timeout (the coding-agent default is `Infinity` unless `timeout_ms` is set, `packages/coding-agent/src/extensions/codemode/execute.ts:484`), 10k-token output budget, 4 concurrent model calls, 3000-token inline declaration budget (`8562bcf66`).
23. **Nested tool-call record: 256 calls, 8 KiB args per call, 32 KiB total, 500 error chars** ([[nested-tool-calls]]) — `packages/coding-agent/src/core/nested-tool-calls.ts:25-30`. Bounds how much codemode activity can bloat the transcript.
24. **Codex WebSocket: 5-min reuse TTL, 55-min max age** ([[http-transport-hardening]], [[session-affinity-cache-routing]]) — `packages/ai/src/api/openai-codex-responses.ts:866-867`. Rotates before the server's ~60-min limit (`23d146261`, #6268).
25. **TUI 16ms render coalescing, 10ms lone-ESC timeout (100ms over SSH)** ([[differential-tui-rendering]]) — `6f5f37f85`, `06ed87167` (#7899). Keeps the UI responsive while streaming without splitting Alt+key sequences.

### Cross-cutting
- **Visible retries over hidden retries**: every SDK-internal retry is 0 (Anthropic `bb445d24f`, provider-wide `8fb1e877c`, Codex `DEFAULT_MAX_RETRIES = 0` `packages/ai/src/api/openai-codex-responses.ts:54`). All waiting happens in the agent loop, with a countdown the user can cancel with Esc.
- **Fixed headroom instead of ratios**: reserve 16384, safety 4096, keep 20000. There is no "compact at 80%" rule anywhere, so 1M-window models compact at about 98%. The Nov-2025 README told users to compact by hand at 80% (`e9935beb5`, "Planned Features").
- **Values duplicated across packages** (truncation, compaction policy, `TOOL_RESULT_MAX_CHARS`, `ESTIMATED_IMAGE_CHARS`, `2^31−1` timeout) between `coding-agent` and `durable`. Drift is likely, and chars/token has already drifted (4 vs 3.5).
- **Locks everywhere** (settings, auth, trust, MCP OAuth, server): almost all `10 × 20ms` sync, or about 30s stale. Several pi processes share `~/.pi`.
- **Absent caps worth tracking across harnesses**: turn cap ([[no-turn-cap]]), bash default timeout ([[no-bash-default-timeout]]), session file size (none found, unverified), read size before decode ([[no-binary-detection-in-read]]).
