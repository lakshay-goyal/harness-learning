---
type: concept
stage: permissions
tier: candidate
aliases: [auth.json, "0o600", bug-report redaction, redactJsonValue, redactUrl, output-guard]
harnesses: [pi]
---
How the harness keeps secrets safe at rest and out of its own outputs: owner-only credential files, indirection so config holds references (`$ENV`, `!command`) instead of literals, redaction in bug reports/diagnostics, repo-supplied config barred from steering credentials — distinct from stream-integrity guards that only look like safety.

## Why
- Credential stores (API keys, rotating OAuth refresh tokens) sit in the home dir of an agent that can `cat` anything; at minimum other local users must not read them.
- Harness-generated artifacts (bug reports, diagnostics, exported sessions) leave the machine; secret-bearing keys and URL credentials must not ride along.
- Repo config that can choose which endpoint receives a stored credential is an exfiltration path (pi forbids `auth.provider` in project MCP config).

## Design space
- **At rest**: plaintext file 0600/dir 0700 (pi) vs OS keychain vs encrypted store.
- **Config values**: literals vs `$ENV`/`${ENV}` templates vs `!shell command` (pi supports all; uppercase strings literal unless `$`-prefixed since `9e9fc7947`).
- **Outbound artifacts**: key-name heuristic redaction + URL userinfo/query scrubbing (pi bug reports) vs allow-list fields only (pi provider diagnostics carry structured fields; values >200 chars dropped for Bedrock).
- **Transcripts**: not redacted (pi) — user told to review before `/export`/`/share`.
- **Credential steering**: repo config may / may not choose credential destination (pi: global-only for MCP provider auth; codemode `models.*` resolves by provider+id only).
- **Sandboxes**: placeholder secrets swapped by an egress proxy ([[tool-only-isolation]]).
- Not in scope: stdout protocol guards (pi `output-guard.ts`) protect JSON/RPC framing, not users.

## Implementations
- [[pi--secret-handling|pi]] — `auth.json` 0600 / dir 0700 under file lock; `resolveConfigValue` `!cmd`/`$ENV`; bug-report `<redacted>` heuristics; structured `diagnostics[]`; temp output files 0600; sessions unredacted.

## Failures
- [[provider-side-history-retention]]
- [[oauth-issuer-mixup-accepted]]
- [[api-key-overrides-subscription-auth]]
- [[concurrent-settings-writes-clobber]]

## Related
[[credential-resolution]] · [[subscription-oauth-auth]] · [[project-trust-gate]] · [[tool-only-isolation]] · [[session-export-share]] · [[errors-as-stream-events]] · [[headless-rpc-mode]] · [[remote-host-trust]]
