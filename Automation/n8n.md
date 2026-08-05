# n8n

**Use for:** Visual webhook bridges (external system -> a code backend), legacy Google Drive -> WhatsApp flows, screenshot workflows. Not the primary backend.
**Status:** Phased out (legacy only)

## Setup & access

- Once you standardize on a code-first automation backend (e.g. Trigger.dev), n8n is best retained only for existing webhook bridges that have no simpler replacement (e.g. form provider -> backend, invoicing provider -> backend, chat platform External Request -> backend).
- New automation work goes to the code backend. Do not start new n8n workflows unless the use case is purely a webhook bridge or visual glue.
- Check workflow JSONs into your repo for version control. Import via UI when re-deploying.
- Public API (`/api/v1/`): requires the API to be enabled in Settings -> n8n API -> Enabled. Without it, every JWT returns 401.
- Instance-level MCP (n8n 2.19.3+ preview): `https://{instance}/mcp-server/http`, Bearer JWT with `aud:"mcp-server-api"`. Narrow scope (whitelisted workflows, no CRUD). Useful for triggering existing flows programmatically.
- n8n Code node sandbox: cannot `require()` Node.js built-ins. Use dedicated nodes instead (Crypto node for HMAC, not the Code node).

## Scars & gotchas

- **Public API keys are per-instance JWTs, a key from another instance 401s everywhere** - a valid-looking n8n API key (correct JWT: `iss:n8n`, `aud:public-api`, non-expiring) can return `401 unauthorized` on every `/api/v1` call. Cause: the key was generated on a **different** n8n instance than the one being called. The JWT is signed with the minting instance's secret, so the target rejects it. Fix: create the key **on the exact instance** you're calling (Settings -> n8n API on that host). Decode the JWT payload (`iss/aud/sub`) to confirm shape, but only the signing instance validates it.

- **`require('crypto')` is blocked in the Code node sandbox** - Attempting to verify HMAC-SHA256 signatures inside a Code node fails: the task-runner sandbox does not allow `require('crypto')`. Fix: use the n8n Crypto node for all HMAC operations.

- **Undocumented HMAC scheme, use a random UUID path as transport secret instead** - Some providers send `x-webhook-signature` (HMAC-SHA256) and `x-webhook-timestamp` headers but do not document which body shape is signed. Without the signing recipe you cannot implement HMAC verification. A random 32-hex-char webhook path is a low-cost interim transport secret; replace it with real HMAC verification once a captured request body lets you reverse-engineer the signing recipe. Note: some providers' own webhook secret field rejects hex-only strings, so you may need an alphanumeric value there.

- **Webflow HMAC header format** - Webflow forms send `x-webhook-signature` (HMAC-SHA256 of the raw body, keyed on the Webflow signing secret) plus `x-webhook-timestamp` (ISO timestamp). Verification pattern: check timestamp is within 5 minutes, then constant-time compare of HMAC. Implement in the Crypto node (not Code node, see above).

- **n8n Public API returns 401 by default** - Wiring an n8n MCP client can waste time on apparent JWT issues. Root cause is often that the instance's Public API was not enabled. Check Settings -> n8n API -> Enabled before debugging auth. If enabling is blocked or slow, export workflow JSON to repo and import via UI instead: it takes ~2 minutes and the JSON in repo is reproducible and reviewable.

- **Two forms on one page need a Code node normalizer** - When one webhook receives payloads from two different forms (e.g. a Lead Form with name + phone, and a Quiz Form with email + quiz fields), branch on a distinguishing field. Example: Lead Form payloads have a `name` field, Quiz Form payloads do not. A Code node normalizer routes each payload to the correct downstream task.

- **Prefer env vars over hardcoding a signing secret in the Crypto node** - a secret hardcoded in the node gets baked into any exported workflow JSON you commit or share, turning the repo into a secret store. Tightly gated editor access can make the practical trust boundary look similar, but env vars keep exported JSON shareable and make rotation cheap. Never hardcode.

- **"Double-`+`" phone formatting bug** - A payload string like `"+{{contact.phone}}"` prefixes a literal `+` onto a phone value that already stores `+972...`. Result: records written as `++972...` (dirty data). Fix: always strip any existing prefix before re-applying formatting: `.replace('+', '')` then prepend `+`. Never blind-concat formatting characters onto values that may already be formatted.

- **Keeping a legacy hostname alive during a platform cutover** - When migrating off a self-hosted n8n instance, keep the old hostname alive as a DNS record so any legacy bridge continues running without interruption, using the same credentials as before. Retire the hostname only once the bridge is fully replaced.

- **Migrating lead capture off n8n to serverless functions + a code backend** - Lead form and funnel entry points can move to serverless functions (e.g. Netlify Functions) plus code-backend tasks. n8n is then retained only for legacy syncs. This confirms phased-out status: new work never starts in n8n; existing bridges stay until they are replaced.

- **The n8n MCP's `updateNode`/`updateSettings` can silently deactivate a live workflow, and the MCP can't re-activate it** - A partial-workflow update via the MCP can save a draft that drops the workflow's active state (the flow goes inactive). `deactivateWorkflow` works, but `activateWorkflow` can return `"Authorization failed - please check your credentials"` (the API key can deactivate but not activate). So never MCP-edit a flow that must stay live without a plan to re-toggle it in the UI. For big or live flows, prefer **editing the exported workflow JSON and re-importing**: it also dodges the MCP<->UI concurrent-edit clobber (a UI save silently reverts MCP changes). Distinguish n8n's draft vs published version when reading state: the live/published version can differ from the draft the MCP returns.

- **1GB droplets stall n8n's API/UI while `/healthz` stays green** - A 1 vCPU / 1GB instance can intermittently take 45-75+ seconds (or time out) on Public API reads, webhook responses, and the UI, while `/healthz` answers in <1s throughout. `/healthz` is a liveness ping only: it proves the process is up, not that the DB/event loop can serve work. Do not use it as evidence an instance is healthy, and treat 1GB RAM as under-spec for any n8n instance carrying live webhook bridges (a 2GB instance degrades far less). Diagnose via instance size before blaming n8n itself.

- **A webhook POST that times out client-side still executes** - During a slowdown, a webhook probe whose HTTP response timed out after 30s (0 bytes received) can still run the workflow (execution recorded). Consequence: a slow n8n makes senders retry on timeout, so the SAME event arrives multiple times even though every delivery executed. Idempotency in the downstream call is what saves you. Never assume "client saw a timeout" means "nothing happened".

- **Public API responses can truncate mid-body under load** - On a starved instance, `POST /workflows/{id}/activate` can return a truncated JSON body (connection cut mid-stream) even though the activation succeeded server-side. On any n8n write that errors oddly, verify actual state (e.g. probe the production webhook URL: 200 "Workflow was started" = active, 404 = inactive) before retrying the write.

- **Error notifications: wire each live flow's `settings.errorWorkflow` + one Error-Trigger workflow** - n8n routes a failed run to the workflow named in its `settings.errorWorkflow`. Build one workflow (Error Trigger -> Telegram node), set it as the error workflow on every live flow. The error trigger isn't webhook/form/chat, so the MCP test tool can't fire it: verify via the executions API (`status: success` on the error workflow = the alert was sent). The error JSON exposes `$json.workflow.name`, `$json.execution.lastNodeExecuted`, `$json.execution.error.message`, `$json.execution.url`. Format the Telegram node with `parse_mode: HTML` (bold labels, `<code>` values, `<pre>` error block), and HTML-escape the dynamic error message (`&`/`<`/`>`) so a stray char can't break the send. A common trap: an alert chat id that is a dead pre-supergroup id will silently drop the alert.

- **n8n-mcp can return NO_RESPONSE while the instances themselves are healthy (healthz 200).** Fall back to direct REST (`/api/v1/executions?status=error`) with the instance API keys. Note: large responses can truncate mid-JSON through some proxies, so page with `limit=5`.

## Conclusions / best practices

- n8n is a bridge layer only. It normalizes and forwards; business logic lives in the code backend.
- New work: start with the code backend. Build an n8n bridge only if the inbound system cannot send directly to the backend with the right headers and body shape.
- Always export and commit workflow JSON to repo. UI-only workflows are a single-instance-failure away from being unrecoverable.
- For HMAC on undocumented webhooks: capture a real inbound request (exec body + headers via the n8n executions API), reverse-engineer the signing recipe, then implement. Do not guess the body shape.
- Use a random-UUID n8n webhook path (32 hex chars) as a low-cost transport secret while HMAC is being figured out. It is not a substitute for real signature verification.
- Idempotency keys belong in the HTTP Request node that calls the backend, not in n8n itself. n8n's Code node computes a per-day or per-event key and passes it as `idempotencyKey` in the task payload.
