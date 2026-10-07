---
type: concept
stage: permissions
tier: candidate
aliases: [auth.json, "0o600", bug-report redaction, redactJsonValue, redactUrl, output-guard, codex-secrets, redact_secrets, "[REDACTED_SECRET]", codex-keyring-store, ILLEGAL_ENV_VAR_PREFIX, NON_INHERITABLE_ENV_VARS, codex-responses-api-proxy]
harnesses: [pi, codex]
---
How the harness keeps secrets safe at rest and out of its own outputs: owner-only credential files, indirection so config holds references (`$ENV`, `!command`) instead of literals, redaction in bug reports/diagnostics, repo-supplied config barred from steering credentials — distinct from stream-integrity guards that only look like safety.

## Why
- Credential stores (API keys, rotating OAuth refresh tokens) sit in the home dir of an agent that can `cat` anything; at minimum other local users must not read them.
- Harness-generated artifacts (bug reports, diagnostics, exported sessions) leave the machine; secret-bearing keys and URL credentials must not ride along.
- Repo config that can choose which endpoint receives a stored credential is an exfiltration path (pi forbids `auth.provider` in project MCP config).

## Design space
- **At rest**: plaintext file 0600/dir 0700 (✔ pi) vs OS keychain vs encrypted store (✔ codex: `age` scrypt files with passphrase in the OS keyring).
- **Config values**: literals vs `$ENV`/`${ENV}` templates vs `!shell command` (pi supports all; uppercase strings literal unless `$`-prefixed since `9e9fc7947`).
- **Outbound artifacts**: key-name heuristic redaction + URL userinfo/query scrubbing (pi bug reports) vs allow-list fields only (pi provider diagnostics carry structured fields; values >200 chars dropped for Bedrock).
- **Transcripts**: not redacted (pi) — user told to review before `/export`/`/share`. codex: regex redaction (`sk-…`, `AKIA…`, `Bearer …`, key/token/secret/password assignments) on displayed command strings in the app-server protocol, not in model context.
- **Child-process env**: inherit everything (pi) vs denylist of harness-internal tokens + optional `*KEY*`/`*SECRET*`/`*TOKEN*` filter (✔ codex; pattern filter off by default since `9fb9ed6cea`) vs proxy-held credentials with dummy values in the child (✔ codex credential broker).
- **Key never readable by the agent**: privileged proxy holds the key in mlocked memory and only forwards one endpoint (✔ codex `codex-responses-api-proxy`); harden the holder process ([[harness-process-hardening]]).
- **Env-file injection**: `.env` may not set harness-control vars (✔ codex forbids `CODEX_*`).
- **Credential steering**: repo config may / may not choose credential destination (pi: global-only for MCP provider auth; codemode `models.*` resolves by provider+id only).
- **Sandboxes**: placeholder secrets swapped by an egress proxy ([[tool-only-isolation]], [[egress-policy-proxy]]).
- Not in scope: stdout protocol guards (pi `output-guard.ts`) protect JSON/RPC framing, not users.

## Implementations
- [[pi--secret-handling|pi]] — `auth.json` 0600 / dir 0700 under file lock; `resolveConfigValue` `!cmd`/`$ENV`; bug-report `<redacted>` heuristics; structured `diagnostics[]`; temp output files 0600; sessions unredacted.
- [[codex--secret-handling|codex]] — `age`-encrypted store + keyring; display redaction `[REDACTED_SECRET]`; `NON_INHERITABLE_ENV_VARS` + `shell_environment_policy`; `.env` cannot set `CODEX_*`; responses-api-proxy (mlock/zeroize); credential broker.

## Failures
- [[provider-side-history-retention]]
- [[oauth-issuer-mixup-accepted]]
- [[api-key-overrides-subscription-auth]]
- [[concurrent-settings-writes-clobber]]
- [[harness-credential-leaks-to-tools]] (codex) · [[error-diagnostics-echo-payload]] (codex) · [[hardening-breaks-child-env]] (codex)

## Related
[[credential-resolution]] · [[subscription-oauth-auth]] · [[project-trust-gate]] · [[tool-only-isolation]] · [[session-export-share]] · [[errors-as-stream-events]] · [[headless-rpc-mode]] · [[remote-host-trust]] · [[egress-policy-proxy]] · [[harness-process-hardening]] · [[protected-workspace-metadata]]
