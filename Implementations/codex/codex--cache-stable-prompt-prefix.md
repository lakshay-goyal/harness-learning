---
type: implementation
harness: codex
concept: cache-stable-prompt-prefix
commit: 622e9e3696
files: [codex-rs/core/src/client.rs:905-941, codex-rs/tools/src/code_mode.rs:117, codex-rs/core/src/context/world_state/environment.rs:35-36, codex-rs/core/src/session/reasoning_effort.rs:1-7, codex-rs/core/src/compact.rs:330-341, codex-rs/codex-api/src/common.rs:279-286]
---
[[cache-stable-prompt-prefix]] in [[codex]].

## Mechanism
- **The prefix is rebuilt on every request but kept byte-identical by deterministic ids.** Prompt-only prefix items get ids `uuidv5(uuidv5(OID, thread_id), payload)` (`codex-rs/core/src/client.rs:908-940`; comment: "These prompt-only items are rebuilt on every request. Hash their visible payloads within the thread so retries and resumed sessions preserve their identity."). Two such items:
  - the base-instructions `developer` message, id suffix `msg`, hashed from the text;
  - for Responses Lite, the `AdditionalTools` developer item, id suffix `at`, hashed from the tools JSON.
- **Base instructions moved out of `instructions` into `input[0]`** (`c9253c4977` 2026-10-05). The cached prefix is now a plain run of input items. A change to the instructions forces a full request rather than a WebSocket delta.
- **Deterministic tool order.** MCP tools were stored in a `HashMap`, which changed their order between turns and "effectively [broke] prompt caching in multi-turn sessions". `ee2ccb5cb6` 2025-08-25 sorts them by name. The pattern persists: code-mode nested tool definitions are sorted and deduplicated by name (`codex-rs/tools/src/code_mode.rs:117-118`).
- **Volatile facts stay out of the system prompt:**
  - The date and timezone sit in the user-role turn environment context (`current_date`, `timezone`; `codex-rs/core/src/context/world_state/environment.rs:35-36`; `90cc4e79a2` 2026-02-26), not in base instructions.
  - The optional wall-clock time arrives as an appended `<current_time_reminder>` developer message ([[current-time-reminder]]).
- **Mid-session setting changes are appended, not rewritten:**
  - World-state section diffs ([[world-state-diff-injection]], [[transcript-carried-system-prompt]]).
  - Personality changes go as a `<personality_spec>` developer message, "so the cached system prompt is untouched" (`8b3521ee77` 2026-01-22).
  - Reasoning-effort changes become `ConfigurationUpdate` items while the request-level effort stays pinned (`codex-rs/core/src/session/reasoning_effort.rs:1-7`; [[cache-preserving-config-update]]).
- **Deltas, not reinjection.**
  - After a command approval only the newly approved prefixes are emitted, instead of the whole permissions block (`1bbfb5cfad` 2026-08-03).
  - Project instructions have a global cap across environments (`85e0661c3b` 2026-08-07).
  - Full-history agent forks drop the parent's multi-agent policy text per item (`663da53823` 2026-08-20).
  - See [[permission-context-reinjected-repeatedly]].
- **Overflow trimming preserves the prefix.** When local compaction overflows, it drops the *oldest* item and retries: "Trim from the beginning to preserve cache (prefix-based) and keep recent messages intact" (`codex-rs/core/src/compact.rs:330-341`).
- **Request body field order** puts the routing fields first (`codex-rs/codex-api/src/common.rs:279-286`). This is for gateways, not the cache.

## Evolution
- `ee2ccb5cb6` 2025-08-25 (#2611): deterministic MCP tool order. Issue #2610.
- `8b3521ee77` 2026-01-22 (#9644): personality updated per turn as a developer message.
- `90cc4e79a2` 2026-02-26 (#12947): local date and timezone in the turn environment context.
- `3b32d861c5` 2026-06-22: environment context migrated to world-state diffs.
- `1bbfb5cfad` 2026-08-03 (#36800): avoid reinjecting permissions after approvals.
- `85e0661c3b` 2026-08-07 (#37424): cap project instructions across environments.
- `663da53823` 2026-08-20 (#39641): sanitize developer context in full-history forks.
- `56a8470aa0` 2026-09-05 (#43110): reasoning-effort pin and configuration updates, with tests for "history prefix and cache-key preservation".
- `c9253c4977` 2026-10-05 (#51156): base instructions as Responses input messages with stable UUIDv5 ids.

## Quirks
- The prompt is not minimal. Catalog prompts are 17–22 KB per model ([[per-model-system-prompt]]). Stability comes from determinism rather than from small size.
- Codex keeps the date *in the prompt history* (a user-role context message), whereas pi removed it entirely ([[no-date-in-prompt]]).

## Versus pi
- [[pi--cache-stable-prompt-prefix|pi]] removes volatile facts and uses a placeholder tool, because Anthropic adds hidden scaffolding when tools change. Codex targets OpenAI's automatic prefix caching only:
  - no breakpoints and no placeholder;
  - investment in byte-identical regeneration (stable ids, sorted tools);
  - append-only context deltas.
- See [[prompt-cache-strategy]].

## Failures
- [[nondeterministic-tool-order-busts-cache]]
- [[permission-context-reinjected-repeatedly]]
- [[cache-key-scoped-to-wrong-identity]]
