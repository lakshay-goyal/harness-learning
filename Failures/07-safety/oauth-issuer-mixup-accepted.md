---
type: failure
concepts: [secret-handling, mcp-integration, subscription-oauth-auth]
harnesses: [pi]
---
**Symptom** — pi's MCP OAuth client exchanged an authorization code from a response naming a *different* issuer than the authorization server the flow started with (OAuth mix-up attack surface: a malicious MCP server/AS could obtain codes/tokens meant for another). Related: step-up authorization requested only the challenge's scopes, so new tokens lost previously granted scopes → endless `insufficient_scope` re-login.

**Root cause** — The client trusted the authorization response without RFC 9207 `iss` validation; granted scope wasn't recorded on tokens.

**Fix · [[pi]]** — `d850edee9` 2026-09-30 "add MCP oauth.authServerMetadataUrl and check RFC 9207 iss" (`packages/mcp/src/oauth/flow.ts`, `discovery.ts`): code exchanged only when `iss` names the flow's AS, or is absent and metadata does not advertise `authorization_response_iss_parameter_supported` (`packages/mcp/CHANGELOG.md:30`); mcp 1.0.0 `stepUpScope()` combines challenge + granted scopes, tokens record granted scope (`packages/mcp/CHANGELOG.md:31,40`). Cross-process refresh serialized `1d74741e1` 2026-09-29.

**Lesson** — An agent that brokers OAuth for third-party tool servers inherits the full OAuth attack surface (mix-up, scope downgrade); validate `iss` and keep scope state.

Related: [[secret-handling]] · [[mcp-integration]] · [[subscription-oauth-auth]] · [[pi--secret-handling|pi]]
