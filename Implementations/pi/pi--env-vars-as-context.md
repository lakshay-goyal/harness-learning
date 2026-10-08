---
type: implementation
harness: pi
concept: env-vars-as-context
commit: b30a6dd77
files: [packages/coding-agent/src/core/tools/bash.ts:186-212, packages/coding-agent/src/core/tools/bash.ts:248-258, packages/coding-agent/docs/environment-variables.md:7-45]
---
[[env-vars-as-context]] in [[pi]].

## Mechanism
- Each shell-tool command gets fresh session facts as env vars, resolved **when the command starts** (`resolveSpawnContext`, `packages/coding-agent/src/core/tools/bash.ts:186-212`):
  - inherited `PI_SESSION_ID/PI_SESSION_FILE/PI_PROVIDER/PI_MODEL/PI_REASONING_LEVEL` deleted first (`:194-198`) — a nested pi can't leak the parent's values;
  - then set from `ctx` when `exposeSessionEnvironment` (default true, `:250`): `PI_SESSION_ID`, `PI_SESSION_FILE` (unset for ephemeral), `PI_PROVIDER`, `PI_MODEL`, `PI_REASONING_LEVEL` (`:199-209`);
  - `spawnHook` may rewrite after (`:210-211`).
- Docs: "Switching models or changing the reasoning level therefore affects the next shell command without restarting Pi. `PI_PROVIDER` and `PI_MODEL` identify the selected Pi model, not a different upstream model that a router may choose internally." (`docs/environment-variables.md:26-32`); not injected into user `!`/`!!` commands (`:49`). `PI_CODING_AGENT=true` process-wide lets children detect pi (`:16`; `cli/setup.ts:6`).
- Prompt carries only a pointer, emitted only when the feature is on: "You can inspect PI_* environment variables for current model and session details." (`bash.ts:47, 257`) → [[dynamic-tool-guidelines]], [[guideline-softening]].
- Why env, not prompt: volatile facts (model switch, reasoning level) would otherwise patch the system prompt every change; the date was removed from the prompt for the same cache reason (`f4e9ca746`) → [[cache-stable-prompt-prefix]]. The model pays a tool call only when it needs the fact.
- Second, vendor-neutral marker `AI_AGENT=pi` set next to `PI_CODING_AGENT=true` by both CLI and RPC entry points (`packages/coding-agent/src/cli/setup.ts:6-7`; `src/rpc-entry.ts:7-8`, which also titles the process `<app>-rpc`) so generic tooling can detect an agent parent; neither marker is set when pi is embedded through the SDK (`docs/environment-variables.md:14-21`). Note the marker is hard-coded `"pi"` even in a `piConfig`-rebranded fork, while `process.title` follows `APP_NAME` (same lines).

## Constants
| name | value | path:line |
|---|---|---|
| vars | PI_SESSION_ID, PI_SESSION_FILE, PI_PROVIDER, PI_MODEL, PI_REASONING_LEVEL | `bash.ts:194-209` |
| `exposeSessionEnvironment` default | true | `bash.ts:250` |
| `PI_CODING_AGENT` | `true` | `docs/environment-variables.md:16` |

## Evolution
- `bb3d7d399` 2026-07-22 (#6967) "expose session metadata to bash tools": vars + imperative guideline "Inspect PI_* environment variables…" + `docs/environment-variables.md` in docs map.
- `4e64de695` 2026-08-06 (#7128): softened to "You can inspect…" — CHANGELOG "reduce unnecessary inspection commands"; commit body "Attempt to address #7128 without closing the issue" (specific commands unverified).
- `80e62761f` 2026-08-24 (#8512): PowerShell tool gets the same vars + identical guideline (deduped).

## Evidence commits
`bb3d7d399`, `4e64de695`, `80e62761f`, `f4e9ca746`.

## Quirks
- Date/time is *not* exposed as a PI_* var; the model must run `date` (no dedicated tool; unverified beyond grep).
- Evals harness model selection also reads `PI_PROVIDER`/`PI_MODEL` from the host env (`packages/evals/src/harness.ts:72-77`) — same names, unrelated use (host config, not model-facing context).

## Failures
- [[imperative-guideline-over-compliance]]
