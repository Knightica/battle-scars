# Green Invoice (Morning / חשבונית ירוקה)

**Use for:** Issuing Israeli tax invoice-receipts (חשבונית מס/קבלה, type 320) after payment; programmatic document generation via REST API
**Status:** Active
**Last validated:** 2026-07-06

## Setup & access

**NEW AUTH (mandatory for anything built after 2026-07-15 - legacy blocked from that date):**

- Token: `POST https://api.morning.co/idp/v1/oauth/token` (note: its own host, NOT `api.greeninvoice.co.il`) with body `{ "grant_type": "client_credentials", "client_id": apiKey, "client_secret": apiSecret }`, `Content-Type: application/json`.
- Response: `{ "accessToken": "<JWT>", "tokenType": "Bearer", "expiresAt": <unix> }`. The token is at **`accessToken`** (not `token`), valid **1 hour**, scoped to the business of the API keys.
- Errors follow OAuth 2.0 (RFC 6749): 400 `invalid_request` (missing grant_type), 400 `unsupported_grant_type`, 400 `invalid_grant` (key expired/revoked/pending), 400 `unauthorized_client` (no API-enabled subscription), 401 `invalid_client` (bad id/secret or blocked account).
- All other endpoints stay on `https://api.greeninvoice.co.il/api/v1` with `Authorization: Bearer <accessToken>`. Sandbox: token at `https://api.sandbox.morning.dev`, API at `https://sandbox.d.greeninvoice.co.il/api/v1`.
- New docs portal: https://developers.morning.co (OpenAPI spec at `/docs/openapi.bundled.json`, v2.0.0 as of 2026-07-06).

Legacy auth (works only until 2026-07-15): `POST https://api.greeninvoice.co.il/api/v1/account/token` with `{ id, secret }`, JWT at `data.token` or the `Authorization` response header.

- Create a document: `POST /api/v1/documents`.
- Credentials: `MORNING_API_KEY` + `MORNING_API_SECRET` in env.

## Scars & gotchas

- **Some accounts add VAT on top of entered prices** - if the account is configured VAT-on-top, do NOT enter gross (customer-paid) amounts. Feed the NET unit price and let Morning compute VAT. Formula: `netUnit = Math.round((gross / (1 + 0.18)) * 100) / 100`. With `rounding: true` in the document body, sub-agora drift is absorbed so the issued total matches the charged amount exactly. Example: ₪100 gross -> net price `Math.round((100 / 1.18) * 100) / 100`. For quantity 2 at ₪200 total gross: quantity 2, price `netUnit(200 / 2)` each. Recommendation: feed net prices in code, do not touch the account-wide "prices include VAT" toggle.

- **Document type 320 requires a payment array with a valid date** - Type 320 (חשבונית מס/קבלה) is a combined invoice + receipt. The receipt leg requires a `payment` array entry. That entry must include `date` set to the actual charge date (ISO `YYYY-MM-DD`). An empty, future, or invalid date returns `errorCode 2426`. Use `new Date().toISOString().slice(0, 10)` at issue time. Also required: `dealType: 1` (regular).

- **Payment type enum** - `type: 3` = credit card (1 = cash, 2 = cheque, 3 = credit card, 4 = bank transfer, 5 = PayPal). Use `type: 3` for PayPlus card payments. The `cardNum` field holds the last 4 digits and Morning renders it as "כרטיס אשראי / XXXX".

- **2026 API migration - legacy auth + legacy base URL blocked from 2026-07-15** - Morning notified integrators that the old infrastructure is blocked starting 15/7/2026. Two breaking changes: (1) token request moved to the OAuth endpoint `POST https://api.morning.co/idp/v1/oauth/token` with `grant_type: "client_credentials"` + `client_id`/`client_secret` (was `POST /account/token` with `{id, secret}`), and the JWT field renamed `token` -> `accessToken` - so both the request AND the response parsing break, not just the URL; (2) base URL `www.greeninvoice.co.il/api` -> `api.greeninvoice.co.il/api`. **Any Greeninvoice integration written or still active after 2026-07-15 MUST use the new auth in "Setup & access" above - do not copy a legacy `getToken()` as-is.** Verified against developers.morning.co OpenAPI v2.0.0.

- **Token location varies - check two places (LEGACY auth only)** - The JWT may appear at `response.data.token` or in the `Authorization` response header depending on the API version. Always try both: `res.data?.token ?? res.headers?.["authorization"]`. Throw if neither is present. On the new OAuth endpoint the field is always `accessToken` in the body.

- **Webhook HMAC signing recipe is undocumented** - Morning webhook payload includes `x-webhook-signature` (64-char hex) and `x-webhook-timestamp` (ISO string) headers, but the signing algorithm is not documented in the official docs. Verification logic was parked pending Morning support response. Accept with caution.

- **One payment fires two webhook events** - A single completed payment emits both `payment/received` and `sale-pages/order-paid`. Without deduplication your handler will create two invoices or two CRM rows. Use the `body.id` field as an idempotency key.

- **Payer details are in body.payer, not body.client** - Webhook payloads put the buyer's name, phone, and email under `body.payer.{name, phone, email}`. The gateway transaction ID is at `body.transactions[0].gatewayTransactionId`.

- **Morning webhook sender user-agent** - `morning webhooks 2.1`. Useful for differentiating Morning calls from PayPlus or other webhook sources in shared endpoints.

## Conclusions / best practices

- Always compute net unit prices in code; never enter gross amounts into Morning if the account is VAT-on-top.
- Always include `rounding: true` in document bodies to handle sub-agora drift.
- For type 320: always include a `payment` array with today's date, the correct gross amount, and `type: 3` for card payments.
- Idempotency key on `body.id` is mandatory if you consume Morning webhooks - one payment fires multiple events.
- Fetch a fresh JWT per task run; on the new OAuth endpoint read it from `accessToken` (1h validity); on legacy, fall back to the `authorization` response header if `data.token` is missing.
- New integrations: always start from the new OAuth auth (see Setup & access), never from a legacy `getToken()`.

## Doc log

- **2026-07-06** - 2026 API migration documented (new OAuth token endpoint at api.morning.co, `accessToken` response field, legacy blocked 2026-07-15). Verified against developers.morning.co OpenAPI v2.0.0. Bumped Last validated.
- **2026-06-25** - Initial consolidation: auth, VAT formula, document type 320 body shape, payment row construction, rounding, and webhook behavior notes.
