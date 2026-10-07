---
type: concept
stage: caching
tier: must-have
aliases: [no date in prompt, __pi_deferred_placeholder__, DEFERRED_TOOL_PLACEHOLDER, leadWithSystem, stable tool descriptions, Baseline System Context, "system rejoin", tool sort, "{{year}}", BaseInstructionsFragment, prefix_namespace, UUIDv5 prefix ids]
harnesses: [pi, opencode, codex]
---
Keep everything before the conversation (system prompt, tool declarations, their order) byte-stable across turns, reloads and days: no timestamps, counts or server lists in it; preseed placeholders for features that would otherwise change hidden scaffolding.

## Why
- Prefix caches hash from the start; one changed byte early (a date, a tool count, a newly connected MCP server in a description) re-bills the whole prompt ([[volatile-system-prompt-prefix]], [[mcp-startup-blocks-and-description-churn]]).
- Provider-hidden scaffolding appears on first use of a feature (Anthropic mid-conversation tool changes) → full miss unless declared from request 1 ([[late-tool-change-rewrites-cache]]).
- Storage order may differ from what providers treat as the prompt (durable: user input committed before the system message) ([[late-tool-change-rewrites-cache]]).

## Design space
- Rebuild prompt each turn with current facts (date/time, model, counts) — pi until 2026-07, abandoned.
- **Remove volatile facts; expose via env/tools** (**pi**: date removed, `PI_*` env).
- **Byte-stable tool descriptions; volatile facts in a patchable prompt section appended as a delta** (**pi**: codemode/tool_search vs `mcp_servers`).
- **Preseed a never-callable placeholder tool** (**pi** Anthropic `__pi_deferred_placeholder__`, "measured: full miss without it").
- **Normalize request order so a system message leads** (**pi-durable** `leadWithSystem`).
- Determinism contract for renderers (pi-durable spec).
- **Deterministic regeneration** (codex): rebuild the prefix every request but make the bytes identical.
  - Prefix items get UUIDv5 ids derived from the thread id and the payload. ✔ codex (`codex-rs/core/src/client.rs:908-940`)
  - Tool lists are sorted by name; hash-map order broke caching (`ee2ccb5cb6`). ✔ codex
- **Volatile facts in a user-role context message, outside the system prompt**
  - Date and timezone in the environment context. ✔ codex
  - Opt-in appended time reminders. ✔ codex
- **Delta-only reinjection**
  - Only newly approved command prefixes; a global cap on project instructions. ✔ codex (`1bbfb5cfad`, `85e0661c3b`)
- **Overflow trimming from the front** (local compaction drops the oldest item) to keep the remaining prefix cacheable. ✔ codex (`codex-rs/core/src/compact.rs:330-341`)
- **Canonical tool order**: sort tool declarations by name at the provider boundary (opencode legacy `83bb216486`) → [[nondeterministic-tool-order-busts-cache]].
- **Fixed number of system blocks**: re-join plugin-added system entries so exactly two cached blocks remain (opencode legacy).
- **Narrow, don't remove**: date → year in a tool description (opencode websearch `{{year}}`); the legacy system `<env>` still carries a daily date.
- **Frozen per-epoch baseline**: render the system context once, store it durably, reuse it verbatim until compaction; changes become appended chronological updates (opencode v2 Context Epoch) → [[transcript-carried-system-prompt]].
- **History transforms must be stable per message**: no request-only rewrite of already-sent text (opencode removed its steering wrapper) → [[ephemeral-history-rewrite-busts-cache]].

## Implementations
- [[pi--cache-stable-prompt-prefix|pi]] — date saga (3 steps), placeholder, stable meta-tool descriptions, leadWithSystem.
- [[codex--cache-stable-prompt-prefix|codex]] — UUIDv5 ids for base instructions and Lite tool items; sorted MCP and code-mode tools; date in environment context; personality and effort changes appended; permission deltas.
- [[opencode--cache-stable-prompt-prefix|opencode]] — legacy: tools sorted by name, system re-joined to ≤ 2 blocks, de-volatilized tool descriptions, but daily date and per-step rebuilt task/MCP descriptions remain; v2: durable Baseline System Context per Context Epoch.

## Failures
- [[duplicated-catalog-in-prompt]]
- [[volatile-system-prompt-prefix]]
- [[mcp-startup-blocks-and-description-churn]]
- [[late-tool-change-rewrites-cache]]
- [[nondeterministic-tool-order-busts-cache]]
- [[permission-context-reinjected-repeatedly]]
- [[cache-key-scoped-to-wrong-identity]]
- [[nondeterministic-tool-order-busts-cache]]
- [[ephemeral-history-rewrite-busts-cache]]
- [[cache-breakpoints-miss-stable-segments]] (opencode: plugin system hook split the system prompt)

## Related
[[transcript-carried-system-prompt]] · [[env-vars-as-context]] · [[minimal-system-prompt]] · [[no-date-in-prompt]] · [[deferred-tool-loading]] · [[mcp-integration]] · [[cache-miss-accounting]] · [[cache-preserving-config-update]] · [[world-state-diff-injection]] · [[current-time-reminder]] · [[prompt-cache-strategy]]

## Tradeoffs
- [[mid-run-user-input]]
- [[prompt-cache-strategy]]
