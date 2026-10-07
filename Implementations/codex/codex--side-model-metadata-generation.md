---
type: implementation
harness: codex
concept: side-model-metadata-generation
commit: 622e9e3696
files: [codex-rs/tui/src/app/thread_title.rs:29, codex-rs/tui/src/app/thread_title.rs:88, codex-rs/tui/src/app/thread_title.rs:265, codex-rs/tui/src/app/thread_title.rs:279, codex-rs/tui/src/app/thread_title.rs:394, codex-rs/tui/src/app/thread_title.rs:415]
---
[[side-model-metadata-generation]] in [[codex]].

## Mechanism
- **Prompt** (one string, `codex-rs/tui/src/app/thread_title.rs:279-289`): "Generate a concise, single-line task title of at most {36} characters and under five words where possible. Start with an imperative verb. Capitalize only the first word unless the user's language, proper nouns, acronyms, or code terms require otherwise. Preserve ticket references exactly. Write in the user's language. Do not use quotes, markdown, or trailing punctuation. Do not answer the request." + "User prompt:\n<first message>" (`:292-305`).
- **/rename variant** adds "Prioritize the current task and latest substantive user request." over the tail of the last `THREAD_TITLE_RECENT_MESSAGES` messages (`:394-411`).
- **Output**: JSON schema `{title: string, minLength 1, maxLength 36}` (`:265-277`); parser rejects output not starting with `{` (`:415-420`).
- **Model routing** (`:88-101`): temporary structured thread on `THREAD_TITLE_MODEL = "gpt-5.6-luna"` with `ReasoningEffort::Low` when provider is OpenAI + ChatGPT account and the model is in the catalog; otherwise the current model.
- Same side-task pattern elsewhere: Guardian API-key reviews on `gpt-5.6-luna` (`c4f42d161a`); async approval classifier `gpt-6-luna` (`codex-rs/ext/guardian-v2/src/async_scorer/decisions.rs:31`) → [[codex--llm-approval-reviewer]]; memory phase-1 extraction at Low effort (`codex-rs/memories/write/src/lib.rs:81-82`) → [[codex--cross-session-memory]].

## Constants
| name | value | path:line |
|---|---|---|
| `THREAD_TITLE_MAX_CHARS` | 36 | `codex-rs/tui/src/app/thread_title.rs:29` |
| `THREAD_TITLE_PROMPT_MAX_BYTES` | 960 | `codex-rs/tui/src/app/thread_title.rs:30` |
| `THREAD_TITLE_RECENT_MESSAGES` | 8 | `codex-rs/tui/src/app/thread_title.rs:31` |
| `THREAD_TITLE_MODEL` | `"gpt-5.6-luna"`, effort Low | `codex-rs/tui/src/app/thread_title.rs:88-101` |

## Evolution
- 2026-08-24 `b3c7e1a47f` "Generate descriptive TUI thread titles (#40492)" — all rules incl. "Do not answer the request" present from the start; same day `5a51caf04d` `/rename` suggestion variant.
- 2026-09-04 `a1294e57f1`: results tracked per originating thread ("Switching threads could therefore leave the originating thread without its generated name").
- 2026-09-12 `b4c864dd64` "Cancel pending thread title generation after manual renames (#45108)".

## Quirks
- Lives in the TUI, not core; whether other clients (desktop/IDE app) generate titles themselves is unverified.

## Versus pi
- No pi implementation recorded in the vault; pi side requests (summaries) use the session model ([[pi--auto-compaction]]).
