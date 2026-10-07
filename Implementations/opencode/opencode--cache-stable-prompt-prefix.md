---
type: implementation
harness: opencode
concept: cache-stable-prompt-prefix
commit: ecc4916b5a
files: [packages/opencode/src/session/llm/request.ts:56-78, packages/opencode/src/session/llm/request.ts:184, packages/opencode/src/session/system.ts:74-85, packages/opencode/src/tool/registry.ts:265-289, packages/opencode/src/tool/websearch.txt:13, packages/core/src/session/context-epoch.ts:40-89, packages/core/src/system-context/builtins.ts:33-39, packages/core/src/session/runner/llm.ts:217-222, CONTEXT.md:130-133]
---
[[cache-stable-prompt-prefix]] in [[opencode]].

## Mechanism

### Legacy runtime — piecemeal measures, prefix still rebuilt every step
- System = ONE joined string [agent or provider prompt, env, references, instructions, MCP instructions, skills, `user.system`] (`packages/opencode/src/session/llm/request.ts:58-66`); if `experimental.chat.system.transform` pushed extra entries but kept the header, the rest is re-joined so there are **at most 2 system blocks** — the first two get cache markers (`packages/opencode/src/session/llm/request.ts:68-78`, `72ebaeb8f7`).
- Tools sorted by name before sending (`packages/opencode/src/session/llm/request.ts:184`, `83bb216486`) → [[nondeterministic-tool-order-busts-cache]].
- Tool descriptions de-volatilized: websearch "The current year is {{year}}" (`packages/opencode/src/tool/websearch.txt:13`, was today's date, `179c40749d`); bash/shell description no longer embeds the project directory (`15a8c22a26`, regressed by `b234370080`, fixed again `38014fe448`) → [[volatile-system-prompt-prefix]].
- Plan/build reminders (non-experimental path) are pushed in memory onto whichever user message is newest (`packages/opencode/src/session/reminders.ts:23-47`): stable across steps of one turn, but when the next user message arrives the earlier one is re-sent without its reminder — one prefix divergence per user turn ([[ephemeral-reminder-injection]]); the queued-message `<system-reminder>` wrapper that rewrote already-sent user text was removed (`f092bafe88`) → [[ephemeral-history-rewrite-busts-cache]].
- **Remaining volatility**: `Today's date: ${new Date().toDateString()}` inside `<env>` (`packages/opencode/src/session/system.ts:83`) changes the system prefix at midnight; instruction files re-read every step; `task` description lists current subagents (`describeTask`) and code-mode description lists the MCP catalog, both rebuilt per step (`packages/opencode/src/tool/registry.ts:265-289`) — an MCP connect or agent change rewrites the tool block ([[late-tool-change-rewrites-cache]], exposure unverified by measurement).

### v2 runtime — Context Epoch with an immutable baseline
- First complete observation of all Context Sources renders a **Baseline System Context**, stored in `session_context_epoch {baseline, snapshot, baseline_seq}` and "reused verbatim across process restarts" (`packages/core/src/session/context-epoch.ts:80-89`, `packages/core/src/session/context-epoch.ts:122-139`; `CONTEXT.md:130-132`): "durably preserves the exact joined text used for the active provider-cache prefix".
- Changes never rewrite the baseline: they append a chronological Mid-Conversation System Message ([[opencode--transcript-carried-system-prompt]]). Date is a source, so a day change is one appended line "Today's date is now: …" (`packages/core/src/system-context/builtins.ts:33-39`).
- Baseline is replaced only after a completed compaction (prefix already invalidated) (`packages/core/src/session/context-epoch.ts:59-70`).
- Remaining prefix breakers (inference from code): `system[0]` is the selected agent's prompt, so an agent switch changes the first system part (`packages/core/src/session/runner/llm.ts:217-219`); the max-steps turn sends `toolChoice: "none"` and no tools (`packages/core/src/session/runner/llm.ts:221-222`), and Anthropic lowering omits `tools` entirely (`packages/llm/src/protocols/anthropic-messages.ts:515-517`).

## Constants
| name | value | path:line |
|---|---|---|
| legacy max system blocks | 2 | `packages/opencode/src/session/llm/request.ts:74-78` |
| legacy date granularity | `toDateString()` (day) | `packages/opencode/src/session/system.ts:83` |

## Evolution
- 2025-12-15 `72ebaeb8f7` rejoin system after plugin hook → [[cache-breakpoints-miss-stable-segments]].
- 2026-02-13 `179c40749d` websearch date → year.
- 2026-03-27 `15a8c22a26` / 2026-03-30 `b234370080` regression / 2026-04-02 `38014fe448` bash description cwd.
- 2026-04-29 `00bb9836a6` instruction order Global → Project → Skills.
- 2026-05-08 `83bb216486` deterministic tool order.
- 2026-06-04 `1af8dafd3e` persist v2 context epochs; 2026-06-19 `f092bafe88` steering wrapper removed; 2026-06-22 `c6ee511485` model/agent switches no longer force a new baseline.

## Quirks / drift
- Cache-stability regressions re-entered through unrelated feature commits (PowerShell support re-added cwd); no test diffs rendered prompts across cwd/date.

Contrast: [[pi--cache-stable-prompt-prefix|pi]] removed the date entirely and pre-declares placeholder tools; opencode legacy still carries a daily date in the system block, v2 freezes the baseline per epoch and appends changes.
