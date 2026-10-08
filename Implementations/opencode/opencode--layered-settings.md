---
type: implementation
harness: opencode
concept: layered-settings
commit: ecc4916b5a
files: [packages/opencode/src/config/config.ts:40-51, packages/opencode/src/config/config.ts:141, packages/opencode/src/config/config.ts:245-247, packages/opencode/src/config/config.ts:371-565, packages/opencode/src/config/managed.ts:8-27, packages/opencode/src/config/variable.ts:33-61, specs/v2/config.md:16, specs/v2/config.md:292-303, specs/v2/provider-policy.md:154-202]
---
[[layered-settings]] in [[opencode]].

## Mechanism
### Legacy runtime (`opencode.json` / `opencode.jsonc`)
- **Merge**: deep merge; `instructions` arrays concatenated and de-duplicated (`packages/opencode/src/config/config.ts:40-51`).
- **Layers**, lowest precedence first (`packages/opencode/src/config/config.ts:371-565`):
  1. remote `<provider url>/.well-known/opencode` for each authenticated provider (`packages/opencode/src/config/config.ts:374`, `packages/opencode/src/config/config.ts:408`);
  2. global `~/.config/opencode/{config.json,opencode.json,opencode.jsonc}` (`packages/opencode/src/config/config.ts:141`, `packages/opencode/src/config/config.ts:272`);
  3. `OPENCODE_CONFIG` file (`packages/opencode/src/config/config.ts:415-417`);
  4. project `opencode.json(c)` from cwd up to the worktree, unless `OPENCODE_DISABLE_PROJECT_CONFIG` (`packages/opencode/src/config/config.ts:420`);
  5. `.opencode/` dirs and `OPENCODE_CONFIG_DIR` (`packages/opencode/src/config/config.ts:432-439`);
  6. `OPENCODE_CONFIG_CONTENT` (`packages/opencode/src/config/config.ts:482-489`);
  7. console org config `<url>/api/config` for the active account (`packages/opencode/src/config/config.ts:510`);
  8. managed dir: `/Library/Application Support/opencode`, `%ProgramData%\opencode` or `/etc/opencode` (`packages/opencode/src/config/config.ts:530-536`; `packages/opencode/src/config/managed.ts:23-27`);
  9. macOS MDM plist domain `ai.opencode.managed` — "override everything" (`packages/opencode/src/config/config.ts:538-546`; `packages/opencode/src/config/managed.ts:8`);
  10. `OPENCODE_PERMISSION` JSON deep-merged into `permission` (`packages/opencode/src/config/config.ts:559-561`).
- **Substitution**: `{env:VAR}` and `{file:path}` in the raw text before parsing (`packages/opencode/src/config/variable.ts:33-61`).
- **Self-write**: a missing `$schema` is inserted by textual replace on the original file (`packages/opencode/src/config/config.ts:245-247`) ([[resolved-secrets-written-back-to-config]]).
- **No trust gate**: project config, `.opencode/plugin(s)` and `.opencode/tool(s)` load on open ([[untrusted-repo-loads-executable-config]]).
- Same dirs also hold `agent(s)/`, `command(s)/`, `skill(s)/`, `plugin(s)/`, `tool(s)/` ([[agent-profiles]], [[plugin-tools]]).

### v2 runtime
- Only `opencode.json(c)` in global, ancestor and `.opencode` dirs; legacy `config.json` unsupported (`specs/v2/config.md:16`; `a4b6047e64`).
- Removals: `tools` boolean map ("lossy compatibility input") in favor of permissions; `command` in favor of skills; `enabled_providers`/`disabled_providers` replaced by ordered policy statements; `small_model` (only used for titles); `experimental.continue_loop_on_deny`; `permission` renamed `permissions` as an ordered array (`specs/v2/config.md:179`, `specs/v2/config.md:210`, `specs/v2/config.md:292-303`, `specs/v2/config.md:382`; `specs/v2/provider-policy.md:23`).
- Policy documents read in reverse so user-global beats repository while statements keep written order; org-managed policy appended last; plugins cannot add statements (`specs/v2/provider-policy.md:154-202`): "a repository cannot silently re-enable something the user denied globally" (`specs/v2/provider-policy.md:162`).
- Permissions authored as ordered arrays ([[permission-rule-order-lost-in-object-config]]).

## Constants
| name | value | path:line |
|---|---|---|
| MDM plist domain | `ai.opencode.managed` | `packages/opencode/src/config/managed.ts:8` |

## Evolution
- 2026-01-17 `052f887a9a` config rewrite leaked substituted secrets.
- 2026-04-25 `66f93035b0`, `a9740b9133`; 2026-05-12 `65368f609d` permission order preserved through decode.
- 2026-05-30 `9583e08be4` v2 location-scoped config loading.
- 2026-06-30 `a4b6047e64` v2 drops legacy config filename.

## Quirks / drift
- Legacy precedence makes repo config override the user's global file (project layer 4 > global layer 2); v2 inverts this for provider policy only.

pi contrast: global + project + CLI with trust-gated project layer and field-level locked writes ([[pi--layered-settings|pi]]).
