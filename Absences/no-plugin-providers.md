---
type: absence
harnesses: [codex]
---
# no-plugin-providers

Model providers are configured only in `config.toml`; no plugin/extension can register a provider, and the project declines to bundle third-party providers.

**What's missing**
- "We do not want to be in the business of adjucating which third-party providers are bundled" (`codex-rs/model-provider-info/src/lib.rs:667-670`).

**Evidence of decision**
- Built-ins limited to OpenAI-family + Responses-compatible local servers (`codex-rs/ollama`, `codex-rs/lmstudio`) and Bedrock (`cefcfe43b9`).

**Implication**
- Contrast pi's [[custom-provider-registration]] from extensions; codex plugins are declarative and cannot add providers ([[no-executable-plugins]]).

Related: [[custom-provider-registration]] · [[provider-breadth]] · [[no-chat-completions-wire]] · [[layered-settings]] · [[Absences]]
