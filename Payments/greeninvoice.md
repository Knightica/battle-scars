# Green Invoice (Morning / חשבונית ירוקה)

**Use for:** Issuing Israeli tax invoice-receipts (חשבונית מס/קבלה, type 320) after payment; programmatic document generation via REST API
**Status:** Active

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

- **Read endpoints worth knowing for reporting** - `POST /documents/search` with an empty body returns every document ever issued, paged (`pageSize` up to 100, response carries `pages`); each item already includes `amount`, `vat`, `amountExcludeVat`, `documentDate`, `status`, `client.name`, so income summaries need no per-document fetch. `POST /expenses/search` filters on the VAT reporting month and its `fromDate`/`toDate` must be the first of the month (`YYYY-MM-01`), not arbitrary dates. `GET /businesses/me` is not in the public OpenAPI spec but works on the new OAuth token (returns `name`, `taxId`, `exemption`, `settings`).

- **The API has no webhook management - re-pointing a webhook is dashboard-only** - `GET /webhooks`, `/account/webhooks` and `/hooks` all return `404`. There is no programmatic way to list, create or re-point a webhook subscription, so any URL change is a manual dashboard action in the account owner's login. Plan for that when a domain migration invalidates a webhook target.

- **One account-level `payment/received` webhook fires for *every* payment, including subscription renewals** - the subscription is per-topic for the whole account, not per product or per payment page. A webhook wired for one-time event purchases can therefore also receive monthly membership renewals on the same fixed-price recurring product (`channel: "recurring-charge"`) months after the original purchase, and a downstream handler that doesn't branch on this will process them as new conversions: junk CRM rows plus a welcome message to people who have been subscribers for months. Branch on `body.channel` and `body.productId`, or gate on your own CRM state ("only greet when the subscription row was *created* this run"), before doing anything user-visible.

- **Legacy `id`/`secret` token auth still worked past the announced cutoff date** - `POST /account/token` with the old API key pair returned `200` and a valid bearer token nearly a month after legacy auth was supposed to die. Useful to know when triaging an old integration, but do not build on it: treat it as borrowed time and migrate to OAuth.

- **A document can never be dated before the last document already issued (errorCode 2405)** - `errorCode 2405` / `התאריך שנבחר עתידי או מוקדם מדי לסוג מסמך זה` does not mean "too old" in the fixed-lookback sense. The valid window is exactly `[documentDate of the most recent document of that type, today]`. Israeli sequential numbering (מספור רציף) requires document numbers to run in date order, so once receipt N exists dated D, nothing can ever be issued before D. Probed live on a fresh account: with a single receipt dated day X, every date between the previous document and day X was rejected, day X through today was accepted, and the day after today was rejected as future. **Consequence for design: issuing late is lossy.** A monthly retainer receipted on the day of payment carries the true date; catching up two months later makes the true date permanently unreachable. Schedule the issuing rather than batching it.

- **`POST /documents/preview` validates a document body without creating it** - Same body, same error codes, returns a base64 PDF in `file` on success. Confirmed that repeated preview calls leave the document count unchanged. This is the only safe way to test whether a date, client, or document shape will be accepted, since a real `POST /documents` is irreversible (a wrong document needs a credit note, not a delete). Always preview before issuing anything whose date or shape is uncertain.

- **The create response is thinner than the read response** - `POST /documents` returns `id`, `number`, `type` and `url`, but **not** `documentDate` or `amount`. Code that reads those straight off the create response gets `undefined`, which then silently propagates into filenames and DB rows (cost a mis-named `...-undefined.pdf` archived to storage). Either fall back to the values you submitted, or re-fetch with `GET /documents/{id}` before using them.

- **Osek patur accounts: type 400 only, and no VAT math at all** - Check `GET /businesses/me` for `exemption: true` plus `settings.documentVatType: 0` / `rowVatType: 0`. An osek patur **cannot** issue a חשבונית מס or a חשבונית מס/קבלה (320) - only a קבלה (**type 400**). On a 400 the money lives in the `payment[]` array and `income[]` is empty, since a receipt acknowledges a payment rather than itemizing a sale. None of the VAT guidance elsewhere applies: no gross-to-net conversion, no `rounding` compensation, feed the actual amount. Do not copy a 320 implementation onto an exempt account.

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

