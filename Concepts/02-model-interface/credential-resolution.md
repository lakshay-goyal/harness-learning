---
type: concept
stage: model-interface
tier: candidate
aliases: [resolveProviderAuth, resolveConfigValue, getApiKey, RuntimeCredentials, credential-resolution-order, stored-credential-owns-provider, credential-chain-precedence, config-value-indirection, per-request-credentials, credential-derived-endpoint, byok-header-suppression, "!command", "$ENV", "<authenticated>"]
harnesses: [pi]
---
Decide which credential a request uses, and where it comes from. The sources in precedence order are:
1. request override,
2. runtime flag,
3. stored credential,
4. config value (literal, `$ENV`, `!command`),
5. ambient env or cloud chains.

The resolved credential can also carry an endpoint, headers or provider-scoped env, and it is resolved fresh for every request.

## Why
- Ambient env discovery misfires:
  - Generic tokens hijack the wrong provider.
  - Sandboxed runtimes have an empty env.
  - Process-global env gets mutated for per-request auth.

  See [[env-credential-discovery-misfires]] and [[process-env-mutated-for-auth]].
- Silently falling back to env after a stored credential fails produces confusing auth.
- Sentinel or placeholder values in the key slot get sent as real keys ([[placeholder-sent-as-api-key]]).
- Indirection syntax is ambiguous: is the value literal, an env var name, or a command? There is also the question of how long command results are cached ([[config-value-indirection-ambiguity]]).
- Auth can carry an endpoint or account config that layers drop ([[credential-scoped-config-dropped]]). SDK credential chains have their own precedence ([[bedrock-credential-and-endpoint-precedence]]).
- Tokens resolved once per run expire mid-run ([[credential-expires-mid-run]]).
- Async init races cache a negative result ([[async-init-race-caches-wrong-value]]).

## Design space
- **Precedence**
  - Env first.
  - A stored credential "owns" the provider: no env fallback after a failed refresh, and none for a credential type without a handler. *pi chose this.*
- **Indirection**
  - Implicit env-name lookup. *pi tried this* and it broke on Windows.
  - Explicit sigils `$VAR`/`${VAR}` and `!cmd`, with `$$`/`$!` escapes. *pi chose this:* 3e9f71744, 9e9fc7947.
- **Command caching**
  - Cache for the process lifetime (`auth.json`).
  - Run per request (`models.json`). *pi uses both,* an undocumented asymmetry; 7a786d88a.
- **Ambient credentials**
  - Copy them into the store.
  - Detect them only, returning a sentinel such as `<authenticated>`, and never persist them. *pi chose this.*
- **Timing**
  - Resolve per run.
  - Resolve per request. *pi chose this:* 1167e8445.
- **Auth payload**
  - A key only.
  - A key plus headers, env, `baseUrl`, and `null` header-deletion markers that suppress SDK placeholder auth on gateways (BYOK confusion).
- **SDK chains**
  - Let the SDK resolve.
  - Disable the SDK's default credential chain and resolve in the harness, e.g. `PiAnthropic._shouldResolveDefaultCredentials → false`.

## Implementations
- [[pi--credential-resolution|pi]] — pi-ai `resolveProviderAuth` (request override, then stored, then ambient), `envApiKeyAuth`, `getApiKeyEnvVars`, provider-scoped env with a `/proc/self/environ` fallback. Coding-agent `RuntimeCredentials`, the provider-composer chain, `resolveConfigValue`, and `getApiKey` per LLM call.

## Failures
- [[env-credential-discovery-misfires]]
- [[process-env-mutated-for-auth]]
- [[async-init-race-caches-wrong-value]]
- [[placeholder-sent-as-api-key]]
- [[config-value-indirection-ambiguity]]
- [[credential-scoped-config-dropped]]
- [[bedrock-credential-and-endpoint-precedence]]
- [[credential-expires-mid-run]]
- [[credential-file-lock-contention]]

## Related
[[subscription-oauth-auth]] · [[http-transport-hardening]] · [[model-catalog]] · [[custom-provider-registration]] · [[layered-settings]] · [[no-sandbox]]
