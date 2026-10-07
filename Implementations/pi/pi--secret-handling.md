---
type: implementation
harness: pi
concept: secret-handling
commit: b30a6dd77
files: [packages/coding-agent/src/core/auth-storage.ts:25, packages/coding-agent/src/core/resolve-config-value.ts:140, packages/coding-agent/src/core/bug-report.ts:19, packages/ai/src/utils/diagnostics.ts:10, packages/ai/src/api/bedrock-converse-stream.ts:415, packages/coding-agent/src/utils/output-files.ts:15, packages/coding-agent/src/extensions/mcp/config.ts:133, packages/coding-agent/src/core/output-guard.ts:45]
---
[[secret-handling]] in [[pi]].

## Mechanism
- **Credential store `~/.pi/agent/auth.json`** (API keys + OAuth tokens): parent dir created `mode 0o700` (`packages/coding-agent/src/core/auth-storage.ts:59`); file written with `AUTH_FILE_WRITE_OPTIONS = {encoding:"utf-8", mode:0o600}` (`:25`) on create and every rewrite (`:65,106,187`); comment: "The mode applies only on creation so administrator-managed modes and ACLs remain intact" (`:24`). Writes under file lock: sync 10 × 20 ms (`:69-71`), async stale 30 s, jittered backoff ≤2 s, abortable (`:120-146`). Docs: "`auth.json` can contain API keys and OAuth tokens. Keep it private and do not commit it." (`packages/coding-agent/docs/providers.md:18`). Plaintext, no keychain.
- **Config indirection** `resolveConfigValue` (`packages/coding-agent/src/core/resolve-config-value.ts:140-150`): `!cmd` → run with configured shell, stdout used, **cached for process lifetime** (`commandResultCache`, `:9`; `:208-219`); `$ENV`/`${ENV}` interpolation; `$$`/`$!` escape literals (`:28-78`). Uppercase bare strings are literals since `9e9fc7947` (2026-06-13, #5661) — previously treated as env refs. `resolveConfigValueOrThrow` errors name the command/env var, not the value (`:229-246`). Used by auth-storage and model registry → see [[credential-resolution]].
- **Credential steering blocked**: project `.pi/mcp.json` may not set `auth` on URL servers — "auth is only allowed in the global mcp.json"; "a repository cannot pick where the credential goes" (`packages/coding-agent/src/extensions/mcp/config.ts:25-26,133-135`). Codemode `models.*`: "A script-supplied baseUrl or headers must never receive the credentials" (resolve by provider+id; catalog strips `headers`) (`packages/coding-agent/src/extensions/codemode/execute.ts:638-641,90-95`).
- **Bug reports** (`packages/coding-agent/src/core/bug-report.ts`, `3c75b2747`, uploaded to Radius `/v1/bug-reports`): `SENSITIVE_KEY = /(?:^|[-_])(api[-_]?key|secret|token|password|passwd|credential|authorization|cookie)(?:$|[-_])/i` applied after camelCase→snake split (`:20-24`); `redactUrl` strips URL userinfo and sensitive query params, recursing into nested `scheme:scheme://` (`:27-48`); `redactJsonValue` replaces sensitive-key values with `"<redacted>"` and URL-scrubs every string (`:51-58`); settings also drop `trackingId`/`deviceId` (`:61-64`); applied to model `baseUrl`/`compat`/`samplingParams`, provider `baseUrl`, extension sources, global+project settings (`:99-177`).
- **Provider failure diagnostics** (`AssistantMessage.diagnostics[]`, "Redacted provider/runtime diagnostics for failures and recoveries", `packages/ai/src/types.ts:564`): shape `{type, timestamp, error{name,message,stack,code}, details}` (`packages/ai/src/utils/diagnostics.ts:10-47`). Bounding is per provider: Bedrock `bedrock_response_failure {status, errorCode (names ending "Exception"), requestId}`, values >`MAX_BEDROCK_DIAGNOSTIC_VALUE_CHARS` 200 dropped not truncated (`packages/ai/src/api/bedrock-converse-stream.ts:412-458`); pi-messages HTTP error bodies truncated to 8192 (`packages/ai/src/api/pi-messages.ts:98-156`). Generic helper keeps message + stack verbatim (no scrubbing) — "redacted" is by field selection.
- **Temp output files** (bash spill, codemode output) created `mode 0o600`, `flag "wx"` (`packages/coding-agent/src/utils/output-files.ts:15,26,33`).
- **Sessions NOT redacted**: "Review sessions before exporting or sharing them. They can contain prompts, tool arguments, command output, file contents, and credentials exposed during the conversation." (`packages/coding-agent/docs/security.md:93`; `docs/usage.md:82`) → [[session-export-share]].
- **Bash env**: inherited `PI_SESSION_*` vars stripped then re-set from ctx (`packages/coding-agent/src/core/tools/bash.ts:186-212`); full `process.env` (incl. provider keys) otherwise passes to commands (inferred from `getShellEnv`, `packages/coding-agent/src/utils/shell.ts:138-150`).
- **`output-guard.ts` is not a safety feature**: `takeOverStdout()` redirects stray `process.stdout.write` to stderr while print/json/rpc owns stdout, keeping a raw writer for protocol frames, with ENOBUFS/EAGAIN retry (`packages/coding-agent/src/core/output-guard.ts:45-70,20-43`; `f1fe49a64`, #2482) — protects JSON/RPC stream integrity from package-manager/extension chatter, not secrets ([[headless-rpc-mode]]).

## Constants
| name | value | path:line |
|---|---|---|
| auth file mode | `0o600` | packages/coding-agent/src/core/auth-storage.ts:25 |
| auth dir mode | `0o700` | packages/coding-agent/src/core/auth-storage.ts:59 |
| `OUTPUT_FILE_MODE` | `0o600` | packages/coding-agent/src/utils/output-files.ts:15 |
| `REDACTED` | `"<redacted>"` | packages/coding-agent/src/core/bug-report.ts:19 |
| `BUG_REPORT_SCHEMA_VERSION` | 1 | packages/coding-agent/src/core/bug-report.ts:18 |
| `MAX_BEDROCK_DIAGNOSTIC_VALUE_CHARS` | 200 | packages/ai/src/api/bedrock-converse-stream.ts:415 |
| auth async lock | stale 30 s, max delay 2 s | packages/coding-agent/src/core/auth-storage.ts:120-121 |

## Evolution
- 2025-12-25 `54018b6cc` AuthStorage/ModelRegistry refactor (0o600 file).
- 2026-03-22 `f1fe49a64` stdout takeover in print/json (#2482).
- 2026-06-13 `9e9fc7947` uppercase config values literal unless `$ENV` (#5661).
- 2026-09-19 `3c75b2747` bug reporting with redaction.
- 2026-09-29 `8562bcf66` project MCP `auth.provider` forbidden.
- 2026-10-04 `d677d0ee7` `OUTPUT_FILE_MODE` for codemode image temp files.

## Evidence commits
`54018b6cc`, `f1fe49a64`, `9e9fc7947`, `3c75b2747`, `8562bcf66`, `d677d0ee7`

## Quirks
- `!command` results cached for the process lifetime — rotating secrets fetched via command go stale until restart.
- Diagnostics persist into session JSONL (message field) — stack traces travel with `/share` exports (inferred).
- Mode 0600 only on create: a pre-existing world-readable `auth.json` stays world-readable (by design per comment `:24`).

## Failures
- [[provider-side-history-retention]] · [[oauth-issuer-mixup-accepted]] · [[api-key-overrides-subscription-auth]] · [[config-value-indirection-ambiguity]] · [[env-credential-discovery-misfires]]
