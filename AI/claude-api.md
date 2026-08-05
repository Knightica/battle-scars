# Claude API (Anthropic)

**Use for:** Email intent classification, contact extraction from CC fields, AI-drafted personalized emails, JSON extraction from unstructured text.
**Status:** Active

## Setup & access

- API key via `ANTHROPIC_API_KEY` (keep it in a local secret store, e.g. macOS Keychain, not in source or a dashboard UI).
- Model IDs and pricing change; consult Anthropic's official docs (docs.anthropic.com) for current model names and rates rather than hardcoding them.

## Scars & gotchas

- **Claude wraps JSON in code fences** - responses sometimes arrive as ` ```json\n{...}\n``` ` instead of bare JSON. `JSON.parse()` on the raw string will throw. Strip the fences before parsing (a simple regex on the response string is sufficient). Treat the response as potentially fence-wrapped even when you instruct Claude not to.

- **Hebrew name transliteration, fallback + retry bump** - a char-by-char Latin fallback (e.g. `transliterateHebrewFallback`) unblocks immediately when the API call fails; bump the Anthropic SDK `maxRetries` from the default 2 to 5; re-run later (e.g. nightly) with the proper API call. Useful for lead-intake and signup tasks.

- **Use a low-cost model (e.g. Claude Haiku) for Hebrew-to-English name transliteration before writing to a downstream system** - when a system requires English names (e.g. a CRM), transliterate Hebrew names at low cost before every write.

## Conclusions / best practices

- Use Claude to classify email intent and extract introduced contacts from the CC field: more robust than regex against varied sender styles.
- Prefer AI-drafted emails over fixed templates for responses that depend on context (e.g., cancellation emails that reference the specific cancellation reason). Results are more personalized and require fewer template variants.
- Always strip ` ```json ` fences before parsing.
- This file covers integration scars only. For model selection and pricing, see Anthropic's official docs.
