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


### codex
- [[agent-writes-its-own-escalation-config]] — workspace-write let the agent write `.git/hooks`, `.codex/`, `.agents/`, `.aws/` that later run with more authority.
- [[sandbox-path-binding-races]] — rename/unlink/symlink races moved protected paths out of their Seatbelt carve-outs.
- [[sandbox-side-channel-syscalls]] — io_uring, AF_VSOCK/WSL interop and mutating fcntls bypassed the obvious filters.
- [[symlinked-roots-escape-sandbox-policy]] — symlinked writable roots evaluated un-canonicalized.
- [[escalation-drops-deny-read]] — approvals / allow rules / safe-list ran commands unsandboxed, dropping deny-read rules.
- [[network-policy-fail-open-paths]] — DNS errors, NO_PROXY metadata range, pre-auth DNS lookups failed open in the egress proxy.
- [[dangerous-command-under-never]] — `rm -rf` ran under `approval_policy = never` because nobody could be asked.
- [[approval-scope-too-broad-in-parallel-batch]] — one approval covered every parallel exec in the turn.
- [[mcp-annotation-defaults-unsafe]] — destructive hint didn't force approval; unannotated MCP tools treated as safe.
- [[model-proposed-rule-too-broad]] — model suggested `["python3"]` / `["bash","-lc"]` as persistent allow rules.
- [[sandbox-failure-misread-as-transient]] — sandbox DNS failures surfaced as ordinary errors; no escalation.
- [[sandbox-network-error-not-escalated]] — prompt didn't name sandbox network-failure signatures.
- [[approval-asked-in-prose]] — model asked for permission in chat / question tool instead of the escalation parameter.
- [[approval-circumvention]] — after a denial the model reached the goal through another tool.
- [[destructive-git-on-user-changes]] — model reverted user edits, `git reset --hard`, amended commits in a dirty worktree.
- [[approval-reviewer-overcautious]] — Guardian denied benign actions; classifier flagged almost everything.
- [[approval-reviewer-trusts-untrusted-content]] — reviewer took tool output / skill text claims as user authorization.
- [[numeric-risk-score-miscalibrated]] — numeric risk scores needed per-prompt thresholds; replaced by categorical labels.
- [[prompt-edit-silently-reverted]] — a Guardian prompt rewrite was rolled back by an unrelated merge five days later.
- [[approval-invalidated-by-new-user-input]] — stale Guardian allows honoured; then any user input aborted pending actions.
- [[hardening-breaks-child-env]] — stripping `LD_*`/`DYLD_*` in the CLI broke every spawned command.
- [[harness-credential-leaks-to-tools]] — internal auth tokens inherited by model-reachable child processes.
- Existing notes with codex blocks: [[hook-error-fails-open]] (PreToolUse hooks fail open; WFP setup non-fatal) · [[untrusted-repo-loads-executable-config]] (git/PATH helpers pre-trust, auto-trust reverted, AGENTS.md gated).

Other groups' failures touching safety concepts: [[bash-descendants-hang-or-lose-output]] · [[windows-process-tree-and-shells]] · [[search-tool-argument-injection]] · [[codemode-sandbox-escape-to-host]] · [[exported-session-xss]] · [[api-key-overrides-subscription-auth]] · [[concurrent-settings-writes-clobber]] · codex: [[approval-wait-surfaces-as-rejection-on-interrupt]] · [[apply-patch-path-and-permission-hazards]] · [[unannotated-mcp-tools-serialized]] · [[error-diagnostics-echo-payload]] · [[client-side-check-wrong-host]] · [[interrupt-kills-background-processes]] · [[provider-stream-ignores-abort]] · [[permission-context-reinjected-repeatedly]]

Back: [[Safety]]
