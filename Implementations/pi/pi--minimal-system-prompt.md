---
type: implementation
harness: pi
concept: minimal-system-prompt
commit: b30a6dd77
files: [packages/coding-agent/src/core/system-prompt.ts:128-193, packages/coding-agent/src/core/system-prompt.ts:88-125, packages/coding-agent/src/core/system-prompt.ts:199-210, packages/ai/src/utils/text.ts:15-40, packages/coding-agent/src/core/tools/read.ts:20-23, packages/coding-agent/src/core/tools/bash.ts:45-48, packages/coding-agent/src/core/tools/edit.ts:43-50, packages/coding-agent/src/core/tools/write.ts:16-19]
---
[[minimal-system-prompt]] in [[pi]].

## Mechanism
- Builder `buildSystemPromptSections()` (`packages/coding-agent/src/core/system-prompt.ts:128-193`) returns ordered named sections; `buildSystemPrompt()` renders them via `getSystemMessageText` exactly as the transcript replays them (`system-prompt.ts:207-210`, `packages/ai/src/utils/text.ts:15-21`, sections joined by blank lines).
- Section order = insertion order: `preamble` (untagged), `tools`, `rules`, `docs`, `addendum`, `project_context`, `skills`, `cwd`, then extension `sections` (`system-prompt.ts:152-186`); every non-preamble section wrapped `<name>\n…\n</name>` (`:188-191`) → [[xml-prompt-boundaries]].
- Harness-authored behavioral text = 1 preamble sentence pair + 2 universal rules ("Be concise in your responses", "Show file paths clearly when working with files", `:122-123`, unchanged since v0 `ffc9be886`). Everything else is contributed: tool snippets/guidelines from each `ToolDefinition` ([[dynamic-tool-guidelines]]), docs pointer ([[self-documentation-pointer]]), context files ([[context-file-hierarchy]]), skills list ([[skill-progressive-disclosure]]), cwd.
- Default selected tools `["read","bash","edit","write"]` (`system-prompt.ts:64`); grep/find/ls off by default → [[minimal-default-toolset]].
- Behavior pushed out of the prompt into: tool descriptions/result notices ([[tool-description-design]], `read.ts:96`, `bash.ts:255`), on-demand docs (`<docs>`), env vars ([[env-vars-as-context]], `bash.ts:47`), codemode reference offloaded to `docs/codemode.md` (`6f1072cc0`).
- **No date/time** anywhere (removed `f4e9ca746`; `git grep 'Current date'` in `packages/coding-agent/src` + `packages/agent/src` → no hits at HEAD) → [[cache-stable-prompt-prefix]], [[no-date-in-prompt]].
- Prompt lives in transcript as `SystemMessage.sections`; later changes are section patches (`diffSystemPromptSections`, `system-prompt.ts:217-229`) → [[transcript-carried-system-prompt]].
- Replace/append/force paths: `customPrompt` (SYSTEM.md / `--system-prompt`) replaces preamble+tools+rules+docs only; `forceSystemPrompt` opaque full replacement (`:199-205`) → [[system-prompt-override]].
- Durable harness reuses the same builder as extension `pi-prompt` with keys `preamble, tools, rules, docs, project_context, skills, cwd` (`packages/coding-agent/src/experimental/durable/prompt.ts:20,61-73`) — no addendum there.

### Current default system prompt at HEAD (rendered; 4 default tools, no context files/skills)
Reconstructed from `system-prompt.ts:155-191` + tool contributions; arrows mark sources. Real rendering separates every section with a blank line (`packages/ai/src/utils/text.ts:20`, join `\n\n`), elided below after `</docs>`.
```
You are an expert coding assistant operating inside pi, a coding agent harness. You help users by reading files, executing commands, editing code, and writing new files.

<tools>
- read: Read file contents
- bash: Execute bash commands (ls, grep, find, etc.)
- edit: Make precise file edits with exact text replacement, including multiple disjoint edits in one call
- write: Create or overwrite files

In addition to the tools above, you may have access to other custom tools depending on the project.
</tools>

<rules>
- Use bash for file operations like ls, rg, find                 ← only if bash/powershell and no grep/find/ls (:108-116)
- Use read to examine files instead of cat or sed.               ← read.ts:22
- You can inspect PI_* environment variables for current model and session details.   ← bash.ts:47
- Use edit for precise changes (edits[].oldText must match exactly)   ← edit.ts:46
- When changing multiple separate locations in one file, use one edit call with multiple entries in edits[] instead of multiple edit calls
- Each edits[].oldText is matched against the original file, not after earlier edits are applied. Do not emit overlapping or nested edits. Merge nearby changes into one edit.
- Keep edits[].oldText as small as possible while still being unique in the file. Do not pad with large unchanged regions.
- Use write only for new files or complete rewrites.             ← write.ts:18
- Be concise in your responses                                   ← :122 (since ffc9be886)
- Show file paths clearly when working with files                ← :123 (since ffc9be886)
</rules>

<docs>
Pi documentation (read only when the user asks about pi itself, its SDK, extensions, themes, skills, or TUI):
- Main documentation: <abs>/README.md
- Additional docs: <abs>/docs
- Examples: <abs>/examples (extensions, custom tools, SDK)
- When reading pi docs or examples, resolve docs/... under Additional docs and examples/... under Examples, not the current working directory
- When asked about: extensions (docs/extensions.md, examples/extensions/), themes (docs/themes.md), skills (docs/skills.md), prompt templates (docs/prompt-templates.md), TUI components (docs/tui.md), keybindings (docs/keybindings.md), SDK integrations (docs/sdk.md), custom providers (docs/custom-provider.md), adding models (docs/models.md), pi packages (docs/packages.md), environment variables (docs/environment-variables.md), MCP servers (docs/mcp.md), codemode scripts and non-LLM models such as classifiers and image models (docs/codemode.md)
- When working on pi topics, read the docs and examples, and follow .md cross-references before implementing
- Always read pi .md files completely and follow links to related docs (e.g., tui.md for TUI API details)
</docs>

<addendum>…APPEND_SYSTEM.md / --append-system-prompt…</addendum>          ← :172
<project_context>
Project-specific instructions and guidelines:

<project_instructions path="/…/AGENTS.md">
…
</project_instructions>
</project_context>                                                         ← :173, :79-86
<skills>
The following skills provide specialized instructions for specific tasks.
Use the read tool to load a skill's file when the task matches its description.
When a skill file references a relative path, resolve it against the skill directory (parent of SKILL.md / dirname of the path) and use that absolute path in tool commands.

<available_skills>
  <skill>
    <name>…</name>
    <description>…</description>
    <location>…/SKILL.md</location>
  </skill>
</available_skills>
</skills>                                                                  ← :174-182, skills.ts:358-388
<cwd>
/home/user/project
</cwd>                                                                     ← :183 (backslashes → "/")
<mcp_servers>…</mcp_servers> + any extension `sections`                   ← :184-186, extensions/mcp/index.ts:1142-1148
```

### Prompt inventory at HEAD (all model-facing prompts)
| # | Prompt | Location | Size |
|---|---|---|---|
| 1 | Default system prompt builder | `system-prompt.ts:128-193` | ~2.7k chars / ~680 tok (4 tools, no ctx/skills) |
| 2 | Section-update renderer | `ai/src/utils/text.ts:15-40` | tiny framing → [[transcript-carried-system-prompt]] |
| 3 | Skills block | `skills.ts:358-388` | 3 lines + ~4/skill → [[skill-progressive-disclosure]] |
| 4 | Context files | `resource-loader.ts:184-270`; render `system-prompt.ts:79-86` | user content → [[context-file-hierarchy]] |
| 5 | SYSTEM.md / APPEND_SYSTEM.md | `resource-loader.ts:1209-1235`; trust `trust-manager.ts:30-39` | → [[system-prompt-override]] |
| 6 | Tool snippets + guidelines | `read.ts:20-23`, `bash.ts:45-48`, `edit.ts:43-50`, `write.ts:16-19`, `grep.ts:35-37`, `find.ts:34-36`, `ls.ts:16-18`, `powershell.ts:18-20` | → [[dynamic-tool-guidelines]] |
| 7 | Tool API descriptions | `read.ts:96`, `bash.ts:255`, `edit.ts:151-152`, `write.ts:52-53`, `grep.ts:78`, `find.ts:78`, `ls.ts:62` | ~40-90 tok each → [[tool-description-design]] |
| 8 | Tool result notices | `read.ts:180-199`, `bash.ts:348-352`, `edit-diff.ts:256-288` | → [[tool-output-truncation]] |
| 9-12 | Compaction system / initial / update / split-turn prompts | `compaction/utils.ts:161-163`, `compaction.ts:507-538`, `:540-579`, `:942-955` | ~60/230/300/120 tok → [[structured-compaction-summary]], [[iterative-summary-update]], [[split-turn-summary]], [[transcript-serialization-for-summary]] |
| 13 | Branch summary prompt + preamble | `branch-summarization.ts:253-286` | ~200 tok → [[branch-summary]] |
| 14 | Summary injection wrappers | `messages.ts:11-25` | ~25 tok → [[auto-compaction]] |
| 15 | Summarizer serializer | `compaction/utils.ts:94-150` | 2000-char tool-result cap |
| 16 | Bug-report summarizer | `bug-report.ts:282-300` (`3c75b2747` 2026-09-19) | ~200 tok; reuses "Do NOT continue the conversation" triple |
| 17 | Anthropic OAuth identity | `ai/src/api/anthropic-messages.ts:1167-1192` | 1 sentence → [[harness-identity]], [[provider-identity-shim]] |
| 18 | Codex empty-instructions fallback | `ai/src/api/openai-codex-responses.ts:557` | "You are a helpful assistant." |
| 19 | codemode description | `extensions/codemode/tool.ts:127-158, 245-298` | budgeted catalog (`DEFAULT_CODEMODE_INLINE_BUDGET=3000`) → [[code-mode]] |
| 20 | `mcp_servers` section | `extensions/mcp/index.ts:155-219, 1142-1148` | ≤4096 chars → [[mcp-integration]] |
| 21 | tool_search description | `extensions/tool-search/tool.ts:215-229` | ~80 tok → [[deferred-tool-loading]] |
| 22 | Durable compaction | `packages/durable/src/harness/compaction.ts:57-90` | ~280 tok |
| 23 | Durable pi-prompt extension | `experimental/durable/prompt.ts:20-74` | reuses builder |
| 24 | Durable prompt planner | `packages/durable/src/harness/prompt.ts:8-158` | no prose |
| 25 | Example subagent prompts | `examples/extensions/subagent/agents/*.md`, via `--append-system-prompt` (`index.ts:300-338`) | 100-190 words → [[subagent-as-subprocess]] |
| 26 | Repo prompt templates | `.pi/prompts/{cl,deslop,is,pr,sa,wr}.md` | 208-1,054 words → [[prompt-template-expansion]] |
| 27 | Other example prompts | `examples/extensions/handoff.ts:20-27`, `qna.ts:14`, `plan-mode/index.ts:207-224`, `preset.ts:20-27`, `pirate.ts:35`, `custom-compaction.ts:55`, `examples/sdk/03-custom-prompt.ts:21`, `12-full-control.ts:42` | — |

## Constants
| name | value | path:line |
|---|---|---|
| default selectedTools | `read, bash, edit, write` | `system-prompt.ts:64` |
| section-name regex | `/^[a-z][a-z0-9_-]*$/` (`preamble` reserved) | `system-prompt.ts:58, 144-148` |
| universal rules | "Be concise in your responses", "Show file paths clearly when working with files" | `system-prompt.ts:122-123` |
| empty tool list text | `(none)` | `system-prompt.ts:159` |
| HEAD prompt size | ~2,706 chars / 371 words ≈ 680 tok (reconstruction, ±15%) | findings 04 |
| v0 size | 696 chars ≈ 175 tok | `ffc9be886` |
| full GPT-5.6 request (default tools + codemode) | ~5,300 → ~3,300 tok | `6f1072cc0` commit msg |

## Evolution
Files: `packages/coding-agent/src/main.ts` (2025-10-17 → 2025-12-09) → `main-new.ts` (`e9f6de7cb`) → `core/system-prompt.ts` (created `1a6a1a8ac` 2025-12-09; 52 commits since). Codex variants lived in `packages/ai/src/providers/openai-codex/prompts/*` (historical: added `1650041a6`, removed `6484ae279`) and `packages/ai/src/constants.ts` (historical: added `6484ae279`, removed `4068bc556`; neither exists at HEAD).

**v0 — `ffc9be886` 2025-10-17 "Agent package + coding agent WIP"** (static `DEFAULT_SYSTEM_PROMPT` in `main.ts`; default provider `google`/`gemini-2.5-flash`; ~117 words / ~175 tok):
```
You are an expert coding assistant. You help users with coding tasks by reading files, executing commands, editing code, and writing new files.

Available tools:
- read: Read file contents
- bash: Execute bash commands (ls, grep, find, etc.)
- edit: Make surgical edits to files (find exact text and replace)
- write: Create or overwrite files

Guidelines:
- Always use bash tool for file operations like ls, grep, find
- Use read to examine files before editing
- Use edit for precise changes (old text must match exactly)
- Use write only for new files or complete rewrites
- Be concise in your responses
- Show file paths clearly when working with files

Current directory: ${process.cwd()}
```

**Full change timeline** (every system-prompt change, quoted text):
| Date | Hash | Change | Class |
|---|---|---|---|
| 2025-11-12 | `9e3e319f1` | Context file (`AGENT.md`/`CLAUDE.md` in cwd) injected as a **user message** "[Project Context from AGENT.md]" | context-file v1 (as message) |
| 2025-11-12 | `dca3e1cc6` | Hierarchical loading: global `~/.pi/agent/` + every ancestor dir, top-most first, each as separate message | monorepo |
| 2025-11-12 | `b1c2c32e2` | "refactor: move context files to system prompt instead of user messages" — `# Project Context` / "The following project context files have been loaded:" / `## <path>`; adds `Current date and time: <locale string>` + `Current working directory:` at the end | context → system prompt; **date added** |
| 2025-11-12 | `a0fa25410` | `--system-prompt` accepts file path; "Project context and datetime still appended automatically" | custom prompt keeps env |
| 2025-11-13 | `c82f9f4f8` | `AGENT.md` → `AGENTS.md` | standard name |
| 2025-11-16 | `0c5cbd006` | Preamble "You are actually not Claude, you are Pi. You are an expert coding assistant…"; + "Your own documentation (including custom model setup) is at: ${readmePath}" / "Read it when users ask about features, configuration, or setup, and especially if the user asks you to add a custom model or provider." | identity override + self-docs |
| 2025-11-20 | `b3d4478b6` (v0.7.23) | + "When summarizing your actions, output plain text directly - do NOT use cat or bash to display what you did" | failure fix [[model-echoes-work-via-shell]] |
| 2025-11-27 | `8b1cca827` (#73) | Identity line removed: "Models now use their native identity instead of being told they are Pi." | rule removed [[forced-model-identity-override]] |
| 2025-11-29 | `186169a82` | **Tool-aware** prompt: per-tool table; conditional "You are in READ-ONLY mode - you cannot modify files or execute arbitrary commands"; "Use bash ONLY for read-only operations (git log, gh issue view, curl, etc.) - do NOT modify any files"; "Prefer grep/find/ls tools over bash for file exploration (faster, respects .gitignore)"; "Use read to examine files before editing" only if read+edit | [[dynamic-tool-guidelines]] born |
| 2025-12-09 | `1a6a1a8ac` | Extracted into `core/system-prompt.ts` (same text) | refactor |
| 2025-12-10 | `7c553acd1` | + "Additional documentation (hooks, themes, RPC, etc.) is in: ${docsPath}" and "…or write a hook." | self-docs grows |
| 2025-12-12 | `09bca9672` (#171) | Skills: "The following skills provide specialized instructions for specific tasks. Use the read tool to load a skill's file when the task matches its description. Skills may contain {baseDir} placeholders…" | [[skill-progressive-disclosure]] |
| 2025-12-17 | `5e5bdadbf` | Docs → topic map "When asked about: custom models/providers (README sufficient), themes (docs/theme.md), skills…, hooks…, custom tools…, RPC (docs/rpc.md)" (`docs/theme.md` historical: merged into `themes.md` in `d79eb99cd`) | self-docs routing |
| 2025-12-19 | `05b7b8133` | Skills standard: XML `<skill><name><description><location>`; `{baseDir}` sentence dropped | spec alignment |
| 2025-12-22 | `42d7d9d9b` | "Use read to examine files before editing" → "…You must use this tool instead of cat or sed." (inside unrelated "before/after session events" commit) | failure fix [[shell-cat-instead-of-read-tool]] |
| 2025-12-31 | `d1465fa0c`, `57dc16d9b`, `84b663276`, `dbdb99c48`, `67a1b9581` | Examples path "(hooks, custom tools, SDK)"; "When asked to create hooks, custom tools, themes, or skills: read the relevant docs AND examples, follow all .md cross-references" → per-topic "When asked to create: hooks (docs/hooks.md, examples/hooks/)…" + "Always read the doc, examples, AND follow .md cross-references before implementing" (`docs/hooks.md` historical: added `7c553acd1`, removed `c6fc08453`) | failure fix (inferred) [[partial-file-read-acted-on]] |
| 2026-01-05 | `c6fc08453` (#454) | + "In addition to the tools above, you may have access to other custom tools depending on the project."; hooks/custom-tools → "extensions" | extension tools exist |
| 2026-01-06 | `59d8b7948` | + "TUI components (docs/tui.md - has copy-paste patterns)" | self-docs |
| 2026-01-07 | `8f5523ed5` | Empty tool list renders "(none)" (`--no-tools`) | edge case |
| 2026-01-08 | `e3dd4f21d` | **Removed** "You are in READ-ONLY mode…" (commit adds extension tool override / `setActiveTools`) | rule removed; rationale inferred (unverified) |
| 2026-01-09 | `f5e6bcac1` → `19b566334` | `prompt = "You are a helpful assistant. Be concise.";` override committed at end of `buildSystemPrompt` inside "Remove Anthropic OAuth support" (with `anthropic-oauth-test-payload.json`); whole commit reverted same day | debug leak (purpose unverified) |
| 2026-01-16 | `6484ae279` (#737) | Preamble → `${PI_STATIC_INSTRUCTIONS}` from `packages/ai/src/constants.ts` (file existed only `6484ae279`→`4068bc556`): "You are pi, an expert coding assistant… Pi specific Documentation: - Main documentation: pi-internal://README.md …", comment "This string is whitelisted by OpenAI and must not change."; dynamic rest sent as developer messages for Codex | provider allowlist coupling → [[harness-identity]] |
| 2026-01-17 | `4068bc556` | Coupling reverted: "You are an expert coding assistant operating inside pi, a coding agent harness. You help users by reading files…"; "Pi documentation (only when the user asks about pi itself, its SDK, extensions, themes, skills, or TUI):"; context header → "Project-specific instructions and guidelines:"; Codex uses system prompt as `instructions` (CHANGELOG `:4110-4111`) | harness identity w/o model identity |
| 2026-01-20 | `b846a4bfc` (#645) | "only when" → "read only when"; "ls, grep, find" → "ls, rg, find"; **removed** "Use bash ONLY for read-only operations…"; custom-tool fallback "Custom tool" | silent removal in ResourceLoader mega-commit |
| 2026-01-23 | `73734a23a` | Only built-in tools listed: "Extension tools are already described via API tool definitions. Removes redundant 'Custom tool' fallback." | dedupe vs API declarations |
| 2026-01-25/26 | `d79eb99cd`, `d2de6d083` | Docs map expanded (prompt templates, keybindings, SDK, custom providers, models, packages); + "When working on pi topics, read the docs and examples, and follow .md cross-references before implementing" + "Always read pi .md files completely and follow links to related docs (e.g., tui.md for TUI API details)" | failure fix (inferred) partial doc reads |
| 2026-03-02 | `bc2fa8d6d`, `8d4a49487` (#1720) | `promptSnippet` + `promptGuidelines` on `ToolDefinition`; dedupe via `addGuideline` Set | guidelines generalized to extensions |
| 2026-03-13 | `4b9e6006f` (#2131) | "Current date and time: Tuesday, …, 14:03:22 GMT" → "Current date: YYYY-MM-DD" — CHANGELOG `:3124` "so prompt prefixes stay cacheable across reloads and resumed sessions" | cache fix #1 [[volatile-system-prompt-prefix]] |
| 2026-03-14 | `671798d67` (#2080) | cwd backslashes → "/" — CHANGELOG `:3091` "Windows backslashes breaking bash tool execution" | [[windows-backslash-cwd-copied-into-shell]] |
| 2026-03-17 | `7817e9b22` (#2285) | Snippets opt-in; no fallback to tool `description` (CHANGELOG `:3037`) | bloat fix |
| 2026-03-22 | `235b247f1` | Snippets/guidelines moved into each built-in `ToolDefinition`; read rule → "Use read to examine files instead of cat or sed." (drops "before editing", "You must"); **silently drops** "When summarizing your actions…do NOT use cat or bash…" (CHANGELOG `:2914` only "Cleaned up `buildSystemPrompt()`") | rule removed |
| 2026-03-27/28 | `20a57e759`, `e773527b3` (#2639) | Edit snippet → "Make precise file edits with exact text replacement, including multiple disjoint edits in one call" + 4 multi-edit guidelines (5 guidelines in dual-mode version, collapsed 1 day later) | tool redesign → [[search-replace-edit]] |
| 2026-04-17 | `f81acc667` (+`7f55605aa` ref #2814) | Date from components not locale (CHANGELOG `:2454` "keeping prompts deterministic across runtimes and locales") | determinism fix |
| 2026-05-16 | `e2fd651eb` (merge `8e6913711`, #4541, external @herrnel) | Custom-prompt path: `# Project Context`/`## path` → `<project_context>` + `<project_instructions path="…">` — "so that agents are less likely to ingest a prompt with inconsistent boundaries" | [[markdown-boundaries-ingested-inconsistently]] |
| 2026-05-18 | `7577d3b8d` (merge `aad8cf660`, #4709) | Same XML boundaries for default prompt | same |
| 2026-05-19 | `48b6510c1` (#4752) | + "When reading pi docs or examples, resolve docs/... under Additional docs and examples/... under Examples, not the current working directory" | [[instruction-relative-paths-resolved-from-cwd]] |
| 2026-05-28 | `1ab289980` (#5132) | **Removed** "Prefer grep/find/ls tools over bash for file exploration (faster, respects .gitignore)" — CHANGELOG `:1806` "avoid preferring unavailable file exploration tools" | [[prompt-names-unavailable-tools]] |
| 2026-07-14 | `f4e9ca746` (#6621, @davidbrai) | **Removed** "Current date: YYYY-MM-DD" — CHANGELOG `:1252` "Fixed system prompt cache invalidation across dates by removing the current date from the default prompt" | cache fix #3 → [[no-date-in-prompt]] |
| 2026-07-22 | `bb3d7d399` (#6967) | bash guideline "Inspect PI_* environment variables for current model and session details."; docs map + environment-variables.md | [[env-vars-as-context]] |
| 2026-08-06 | `4e64de695` (#7128) | → "You can inspect PI_* environment variables…" — CHANGELOG `:759` "Softened … in an attempt to reduce unnecessary inspection commands" | [[imperative-guideline-over-compliance]] |
| 2026-08-10 | `3dd4623ee` (#7887) | Trailing newline after cwd in custom path — "custom system prompts concatenating the current working directory with later appended prompt content" | formatting bug |
| 2026-08-24 | `80e62761f` (#8512) | PowerShell variants of the shell rule (`:108-116`) | platform |
| 2026-09-03 | `1d6dbf9e3` (#8552) | Skills shown when only bash: "Use bash to load a skill's file…" | [[skills-hidden-when-read-tool-absent]] |
| 2026-09-16 | `9e05370b2` (#9548, Armin Ronacher) | **Structured sections** `preamble`, `<tools>`, `<rules>`, `<docs>`, `<addendum>`, `<project_context>`, `<skills>`, `<cwd>`; "Available tools:"/"Guidelines:"/"Current working directory:" headers replaced by tags; stored as `SystemMessage.sections`; changes become patches — "makes system prompt text and tool changes part of the transcript rather than silently rewriting its starting conditions… preserve cached prompt prefixes where the upstream supports it" | cache + resumability → [[transcript-carried-system-prompt]] |
| 2026-09-16 | `e4c75a732` | `forceSystemPrompt` = opaque replace message (previously models with native mid-convo system messages "kept the original prompt as their leading system prompt and received the forced one as a later update") | [[forced-system-prompt-applied-as-late-update]] |
| 2026-09-29 | `8562bcf66` | + "MCP servers (docs/mcp.md)" in docs map | MCP added (reverses "No MCP") |
| 2026-10-01 | `6f1072cc0` | + "codemode scripts and non-LLM models such as classifiers and image models (docs/codemode.md)"; "With the default tools a GPT-5.6 request drops from about 5,300 to 3,300 tokens." | size reduction via docs offload |
| 2026-10-05 | `c30840c2e` (#10343) | Hidden tools excluded from rules/tool list; skills hint "indirect" | consistency |

**Removed rules (what · added · removed · why)**
| Rule (quoted) | Added | Removed | Why |
|---|---|---|---|
| "You are actually not Claude, you are Pi." | `0c5cbd006` 2025-11-16 | `8b1cca827` 2025-11-27 | "Models now use their native identity instead of being told they are Pi." (#73) |
| "Always use bash tool for file operations like ls, grep, find" | `ffc9be886` | `186169a82` 2025-11-29 | became conditional; later "ls, rg, find" (`b846a4bfc`) |
| "You are in READ-ONLY mode - you cannot modify files or execute arbitrary commands" | `186169a82` | `e3dd4f21d` 2026-01-08 | no message; coincides with extension tool overrides (inferred) |
| "Use bash ONLY for read-only operations (git log, gh issue view, curl, etc.) - do NOT modify any files" | `186169a82` | `b846a4bfc` 2026-01-20 | silent; prompt-only read-only was unenforceable w/o permission system (inferred; cf. reviewer.md "Assume tool permissions are not perfectly enforceable") |
| "Documentation: Your own documentation … Read it when users ask about features…" | `0c5cbd006` | `5e5bdadbf` / `4068bc556` | replaced by topic map + "read only when the user asks about pi itself" |
| `PI_STATIC_INSTRUCTIONS` "You are pi, an expert coding assistant… pi-internal://README.md" | `6484ae279` 2026-01-16 | `4068bc556` 2026-01-17 | "use the default system prompt directly for Codex instructions" (CHANGELOG `:4111`) |
| Codex bridge `<critical_rule>` APPLY_PATCH/UPDATE_PLAN DOES NOT EXIST | `1650041a6` 2026-01-04 | `6484ae279` 2026-01-16 | Codex compatibility finalized (#737) |
| Skills "{baseDir} placeholders" sentence | `09bca9672` | `05b7b8133` | standard compliance; later explicit relative-path rule (`3c687b427`, `5d6a7d6c3` #1171) |
| "When summarizing your actions, output plain text directly - do NOT use cat or bash to display what you did" | `b3d4478b6` 2025-11-20 | `235b247f1` 2026-03-22 | silent during refactor |
| "Use read to examine files before editing. You must use this tool instead of cat or sed." | `42d7d9d9b` | `235b247f1` | softened to "Use read to examine files instead of cat or sed." |
| Tool "Custom tool" fallback / description fallback | `b846a4bfc` | `73734a23a`, `7817e9b22` (#2285) | "Extension tools are already described via API tool definitions" |
| "Prefer grep/find/ls tools over bash for file exploration (faster, respects .gitignore)" | `186169a82` | `1ab289980` 2026-05-28 | preferred unavailable tools (#5132) |
| "Current date and time: …" → "Current date: …" | `b1c2c32e2` 2025-11-12 | `4b9e6006f`, `f4e9ca746` 2026-07-14 | cache invalidation (#2131, #6621) |
| Bash "Commands run with a 30 second timeout." | `ffc9be886` | `29900ce64` 2025-11-12 | timeout optional |
| Edit dual mode (oldText/newText vs edits[]) | `20a57e759` | `e773527b3` +1 day | invalid tool calls (#2639) |
| Compaction "You are performing a CONTEXT CHECKPOINT COMPACTION" + free-form list | `6c2360af2` | `ac71aac09` | EXACT structured format |
| Turn-prefix "PREFIX… SUFFIX" wording | `a38e61909` | `d192bd6dc` 2026-09-22 | Fable refusals (#9652/#9908) |
| Summarizer "AI coding assistant" | `3c6c9e52c` | `72fd91135` | non-coding agents (#5401) |
| README "Philosophy: No MCP / No Sub-Agents / No Built-in To-Dos ("To-do lists confuse models more than they help") / No Planning Mode / No Permission System (YOLO) / No Background Bash" | `e27593798` 2025-12-09 | section gone after `25cc5c7bf` 2026-09-22; MCP shipped `8562bcf66` | ideology partially reversed; README now "skips features like sub-agents and plan mode" (`README.md:19`) → [[no-builtin-mcp-reversed]] |

**Prompt size**
| Point | Evidence | ≈ size (4 tools, no AGENTS.md/skills) |
|---|---|---|
| v0 2025-10-17 | `ffc9be886` | 696 chars ≈ 175 tok |
| 2025-12-09 | `1a6a1a8ac` template 493 + guidelines 445 chars + date/cwd | ≈ 1.1k chars ≈ 270 tok |
| 2026-01-26 | `d2de6d083` template 1,172 chars | ≈ 1.8k chars ≈ 450 tok |
| HEAD | reconstruction | 2,706 chars / 371 words ≈ 680 tok |
| HEAD full request + codemode | `6f1072cc0` | ~3,300 tok whole GPT-5.6 request (was ~5,300) |

Verdict: <1k tok of harness text (true), but grown ~4× since v0 — ~45% `<docs>` self-docs, ~20% edit guidelines; behavioral core barely changed. README only says "Pi is a minimal, extensible agent harness" (`packages/coding-agent/README.md:15`); "tiny" token count is folklore (unverified in-repo).

## Evidence commits
`ffc9be886`, `9e3e319f1`, `dca3e1cc6`, `b1c2c32e2`, `a0fa25410`, `c82f9f4f8`, `0c5cbd006`, `b3d4478b6`, `8b1cca827`, `186169a82`, `1a6a1a8ac`, `7c553acd1`, `09bca9672`, `5e5bdadbf`, `05b7b8133`, `42d7d9d9b`, `d1465fa0c`, `57dc16d9b`, `84b663276`, `dbdb99c48`, `67a1b9581`, `c6fc08453`, `59d8b7948`, `8f5523ed5`, `e3dd4f21d`, `f5e6bcac1`, `19b566334`, `6484ae279`, `4068bc556`, `b846a4bfc`, `73734a23a`, `d79eb99cd`, `d2de6d083`, `bc2fa8d6d`, `8d4a49487`, `4b9e6006f`, `671798d67`, `7817e9b22`, `235b247f1`, `20a57e759`, `e773527b3`, `f81acc667`, `7f55605aa`, `e2fd651eb`, `8e6913711`, `7577d3b8d`, `aad8cf660`, `48b6510c1`, `1ab289980`, `f4e9ca746`, `bb3d7d399`, `4e64de695`, `3dd4623ee`, `80e62761f`, `1d6dbf9e3`, `9e05370b2`, `e4c75a732`, `8562bcf66`, `6f1072cc0`, `c30840c2e`, `e27593798`, `25cc5c7bf`.

## Quirks
- `f5e6bcac1` debug override shipped (briefly) in main: shows the prompt builder has no test pinning its output at that time (unverified whether tests exist now).
- Silent rule drops inside unrelated commits (`42d7d9d9b` added, `235b247f1` removed, `b846a4bfc` removed) — prompt changes are not consistently in CHANGELOG; mining needs `git log -S`.
- Open: why "When summarizing your actions… do NOT use cat or bash" was dropped (`235b247f1`) — intentional or accidental (unverified).
- Open: why "Use bash ONLY for read-only operations" removed in `b846a4bfc` (#645 not fetched).
- Open: with no date in prompt, date-dependent tasks rely on the model running `date` via bash; no extension/tool supplies it (grep found none, unverified beyond grep).
- Token counts are chars/4 on hand reconstruction (±15%); running `buildSystemPrompt` not done.
- Durable `pi-prompt` drops `addendum` (APPEND_SYSTEM) — KEYS list (`experimental/durable/prompt.ts:20`).

## Failures
- [[model-echoes-work-via-shell]]
- [[shell-cat-instead-of-read-tool]]
- [[forced-model-identity-override]]
- [[windows-backslash-cwd-copied-into-shell]]
- [[markdown-boundaries-ingested-inconsistently]]
- [[partial-file-read-acted-on]]
- [[prompt-names-unavailable-tools]]
- [[imperative-guideline-over-compliance]]
- [[volatile-system-prompt-prefix]]
- [[forced-system-prompt-applied-as-late-update]]
