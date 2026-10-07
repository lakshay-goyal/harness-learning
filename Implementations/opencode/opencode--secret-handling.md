---
type: implementation
harness: opencode
concept: secret-handling
commit: ecc4916b5a
files: [packages/opencode/src/auth/index.ts:10, packages/opencode/src/auth/index.ts:59-61, packages/opencode/src/auth/index.ts:78-88, packages/opencode/src/mcp/auth.ts:37-38, packages/opencode/src/mcp/auth.ts:80, packages/opencode/src/config/variable.ts:33-61, packages/opencode/src/config/config.ts:245-247, packages/opencode/src/agent/agent.ts:128-134, packages/opencode/src/share/share-next.ts:274-300, packages/opencode/src/control-plane/workspace.ts:529-536, packages/llm/src/route/executor.ts:39-56, packages/llm/src/route/executor.ts:203-206, specs/v2/schema-changelog.md:395-405]
---
[[secret-handling]] in [[opencode]].

## Mechanism
### Legacy runtime
- **At rest**: provider credentials in `<data>/auth.json`, written with mode `0o600` on every set/remove (`packages/opencode/src/auth/index.ts:10`, `packages/opencode/src/auth/index.ts:78-88`). MCP OAuth tokens in `<data>/mcp-auth.json`, `0o600`, under a named file lock (`packages/opencode/src/mcp/auth.ts:37-38`, `packages/opencode/src/mcp/auth.ts:80`). Plaintext; no keychain.
- **Env override**: `OPENCODE_AUTH_CONTENT` (JSON) replaces the auth file when set (`packages/opencode/src/auth/index.ts:59-61`). This is how credentials reach workspace targets (below).
- **Config indirection**: `{env:VAR}` and `{file:path}` substituted into config text at load (`packages/opencode/src/config/variable.ts:33-61`). The loader inserts a missing `$schema` by textual replace on the original text (`packages/opencode/src/config/config.ts:245-247`), never by re-serializing the substituted object ([[resolved-secrets-written-back-to-config]]).
- **Secret files vs the model**: no hard block; default permission rules `read: {*.env: ask, *.env.*: ask, *.env.example: allow}` (`packages/opencode/src/agent/agent.ts:128-134`). Applies to the `read` permission only; shell `cat .env`, `grep` and an earlier "always" on any read bypass it ([[secret-guard-bypassed-by-other-tools]]).
- **Share upload**: `/share` (or `share: "auto"`) POSTs the full session: info, every message, **every part** including tool inputs/outputs and file contents read, the snapshot diffs and model metadata (`packages/opencode/src/share/share-next.ts:274-300`). No redaction (no `redact` in the file); the only transcript redaction is the opt-in, wholesale `opencode export --sanitize` (`packages/opencode/src/cli/cmd/export.ts:11-17`; `3695057bee`). Default is manual; `OPENCODE_DISABLE_SHARE` kills every path ([[session-export-share]]).
- **Workspace targets**: creating a control-plane workspace passes `OPENCODE_AUTH_CONTENT = JSON.stringify(auth.all())` (every stored provider credential) plus `OTEL_EXPORTER_OTLP_HEADERS` to the adapter's environment (`packages/opencode/src/control-plane/workspace.ts:529-536`). Plugin adapters can provision anything ([[credentials-forwarded-to-execution-target]]).

### v2 runtime
- **Provider error redaction** (`packages/llm/src/route/executor.ts:39-56`): one sensitive-name regex (`authorization|api[-_]?key|access[-_]?token|refresh[-_]?token|id[-_]?token|token|secret|credential|signature|x-amz-signature`) applied to header names, JSON body fields and query keys (plus short `key`/`sig` query keys) → `<redacted>`. Provider error bodies included in messages only when ≤ 500 chars (`packages/llm/src/route/executor.ts:203-206`).
- **Public DTOs**: internal catalog records "may contain credentials or provider-specific request material and must not cross the public HTTP serialization boundary" → explicit public DTOs without headers, bodies, API settings (`specs/v2/schema-changelog.md:395-405`) ([[internal-errors-leak-over-wire]]).

## Constants
| name | value | path:line |
|---|---|---|
| auth file mode | `0o600` | `packages/opencode/src/auth/index.ts:79` |
| MCP auth file mode | `0o600` | `packages/opencode/src/mcp/auth.ts:80` |
| provider error body cap | 500 chars | `packages/llm/src/route/executor.ts:204` |
| redaction marker | `<redacted>` | `packages/llm/src/route/executor.ts:39` |

## Evolution
- 2025-12-22 `60db171b44` `.env` read block narrowed so `.envrc` is not blocked.
- 2026-01-04 `3611260405` hardcoded `.env` refusal in read.ts replaced by default permission rules.
- 2026-01-17 `052f887a9a` config loader stopped writing `{env:…}`-resolved values back into `opencode.json`.
- 2026-02-27 `c12ce2ffff`, 2026-03-03 `7f37acdaaa` remote workspaces and adapter interface (credential forwarding).

## Quirks / drift
- `SECURITY.md:31` puts "LLM provider data handling" out of scope; transcripts, including any secrets the agent read, go to the provider and, when shared, to `opncd.ai` unredacted.
- No secret scrubbing of tool output before it enters context.

pi contrast: same 0600 plaintext store and unredacted transcripts, but pi redacts its bug reports and bars repo config from steering credentials ([[pi--secret-handling|pi]]).
