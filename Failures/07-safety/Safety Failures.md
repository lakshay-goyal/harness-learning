---
type: group
group: 07-safety
---
Failures whose primary concept is in [[Safety]].

- [[hook-error-fails-open]] — remote-exec routing hook error fell back to running the user's command on the local shell.
- [[side-door-input-bypasses-hooks]] — RPC inputs, custom tools and error results skipped the extension hooks used as the policy layer.
- [[pre-tool-hook-sees-stale-state]] — `tool_call` gates read session state missing the assistant message being judged.
- [[untrusted-repo-loads-executable-config]] — cloning a repo and starting pi ran its `.pi/extensions` with user privileges.
- [[trust-scope-includes-agent-config-dir]] — running from `$HOME` treated the agent's own config as untrusted project input.
- [[repo-config-disables-sandbox-plugin]] — (latent) repo `.pi/sandbox.json` can disable a globally installed sandbox plugin; not trust-gated.
- [[liveness-timeout-counts-frames-not-bytes]] — a slow large frame could trip the remote daemon's 30 s kill-all dead-man switch.
- [[remote-binary-trusted-by-version-name]] — remote daemon identified by version name, not content hash.
- [[internal-errors-leak-over-wire]] — internal exceptions serialized to remote protocol clients.
- [[auth-at-message-layer]] — bearer auth as a protocol message instead of at the transport.
- [[provider-side-history-retention]] — OpenAI/Azure Responses stored conversations server-side by default.
- [[oauth-issuer-mixup-accepted]] — MCP OAuth exchanged codes from a different issuer; step-up dropped granted scopes.

## Permission rules (opencode)
- [[permission-rule-order-lost-in-object-config]] — last-match-wins rules lost their written order through object-shaped config.
- [[declined-action-retried]] — a user decline came back as a tool error and the model retried; decline now ends the run.
- [[permission-feedback-lost-in-tool-error]] — (latent) v2 leaf tools collapse correction feedback and policy blocks into "Unable to …".
- [[approval-wait-without-responder]] — headless runs, subagent asks and agent-less sessions waited forever on approvals; GitHub run still has no responder.
- [[error-message-leaks-filtered-catalog]] — legacy skill not-found error lists every skill, including ones the agent's permissions deny; v2 fixed.

## Secrets (opencode)
- [[secret-guard-bypassed-by-other-tools]] — `.env` ask rule on `read` only; shell, grep and a prior "always" bypass it.
- [[resolved-secrets-written-back-to-config]] — `$schema` insertion re-serialized `{env:…}`-resolved config into `opencode.json`.
- [[credentials-forwarded-to-execution-target]] — workspace creation exports every stored credential to adapter-provisioned targets.

Other groups' failures touching safety concepts: [[read-only-mode-bypass-via-subagent]] · [[read-only-mode-bypassed-via-shell]] · [[dead-hook-in-public-api]] · [[bash-descendants-hang-or-lose-output]] · [[windows-process-tree-and-shells]] · [[search-tool-argument-injection]] · [[codemode-sandbox-escape-to-host]] · [[exported-session-xss]] · [[api-key-overrides-subscription-auth]] · [[concurrent-settings-writes-clobber]]

Back: [[Safety]]
