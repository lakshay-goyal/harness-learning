---
type: absence
harnesses: [pi]
---
# no-date-in-prompt

**What's missing**
- The default system prompt has no current date or time. `git grep 'Current date'` over `packages/coding-agent/src` and `packages/agent/src` → no hits at `b30a6dd77`.
- The prompt ends with `<cwd>` and optional `<mcp_servers>` / extension sections (`packages/coding-agent/src/core/system-prompt.ts:183-186`).

**Evidence of decision**
- `f4e9ca746` (2026-07-14, David Brailovsky, fixes #6621) "remove current date from system prompt". Deletes the `now`/`year`/`month`/`day` computation and `prompt += \`\nCurrent date: ${date}\``.
- CHANGELOG (`packages/coding-agent/CHANGELOG.md:1252`): "Fixed system prompt cache invalidation across dates by removing the current date from the default prompt".

**History** (3-step saga, all for the prompt cache)
| Date | Hash | Change | Why |
|---|---|---|---|
| 2025-11-12 | `b1c2c32e2` | Context files moved into the system prompt; adds `Current date and time: <locale string>` + `Current working directory:` at the end | env in prompt |
| 2025-11-12 | `a0fa25410` | `--system-prompt` file path; "Project context and datetime still appended automatically" | custom prompt keeps env |
| 2025-12-11 | `2a0f23928` | mom (Slack bot) precedent: "remove dynamic timestamp from system prompt for better cache hits" | cache |
| 2026-03-13 | `4b9e6006f` (#2131) | Time dropped → `Current date: YYYY-MM-DD`; CHANGELOG `:3124` "so prompt prefixes stay cacheable across reloads and resumed sessions" | cache fix 1 |
| 2026-04-17 | `f81acc667` (#2814) | Date built from components, not locale; "keeping prompts deterministic across runtimes and locales" (CHANGELOG `:2454`) | determinism |
| 2026-07-14 | `f4e9ca746` (#6621) | Date removed entirely | cache fix 3 → absence |

**Opt-in replacement**
- The model can run `date` via bash. The session environment is exposed as `PI_*` env vars rather than prompt text (bash guideline "You can inspect PI_* environment variables for current model and session details", `packages/coding-agent/src/core/tools/bash.ts:47`, same in `powershell.ts:20`; gated by `exposeSessionEnvironment`, `bash.ts:221,250`) → [[env-vars-as-context]]. No date env var is documented (unverified).
- `APPEND_SYSTEM.md` / the `before_agent_start` hook / `systemPromptOptions` can add a date ([[system-prompt-override]]; `examples/extensions/prompt-customizer.ts`). Doing so re-breaks the daily cache prefix.
- Open question: no extension or tool in coding-agent/agent src provides the date (`04-prompts` open question 5).

**Implication**
- Anything volatile in the system prompt costs a full cache miss every session/day. pi treats the date as volatile, so models must fall back on training-cutoff assumptions or ask bash.
- Part of a broader rule ([[cache-stable-prompt-prefix]]): volatile MCP server lists moved to an appended `mcp_servers` section (#10212). Section changes are sent as mid-conversation system updates, not prefix rewrites (`9e05370b2`; [[transcript-carried-system-prompt]]).

Related: [[cache-stable-prompt-prefix]] · [[minimal-system-prompt]] · [[env-vars-as-context]] · [[transcript-carried-system-prompt]] · [[system-prompt-override]] · [[Absences]]
