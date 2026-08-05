# PayPlus

**Use for:** Hosted-page payment links (standing orders, one-off charges) for Israeli merchants; IPN webhook for server-side payment confirmation
**Status:** Active
**Last validated:** 2026-08-01

**TL;DR:** Auth header is `JSON.stringify({api_key, secret_key})`, not Bearer. The IPN webhook body is **sparse and unsigned - never trust it**: always phone home to `/PaymentPages/ipn` with the UID to get real status, and store *that* UID, not the dashboard row ID. Use **exactly one** callback channel (dashboard OR per-request `refURL_callback`), never both, or you double-issue invoices. Ship a manual "mark paid" button from day one.

## Setup & access

- API key + secret live on the merchant account. Set as `PAYPLUS_API_KEY` + `PAYPLUS_SECRET_KEY` in env.
- Also need `PAYPLUS_PAYMENT_PAGE_UID` (the specific hosted-page UID from the PayPlus dashboard).
- Base URL: `https://restapi.payplus.co.il/api/v1.0`
- Do NOT use the `payplus-il` npm SDK - it is stale and has no TypeScript types. Use direct `fetch` + JSON.
- Generate a link: `POST /PaymentPages/generateLink`
- Verify a transaction: `POST /PaymentPages/ipn`

## Scars & gotchas

- **API permission is account-level, not code-level** - "THIS COMPANY DONT HAVE THE PERMISSION TO USE THE API" means the merchant account has not had API access enabled by PayPlus. Contact PayPlus support to flip the flag. The code is fine; no redeploy needed once support enables it.

- **Authorization header is JSON-stringified, not Bearer** - The `Authorization` header must be `JSON.stringify({ api_key: apiKey, secret_key: secretKey })`. This is not a standard Bearer token. Getting this wrong gives an auth error with no other hint.

- **IPN body is sparse and unsigned - never trust it alone** - The server-to-server IPN POST body does not contain authoritative status or the canonical subscription UID. Always phone home to `POST /PaymentPages/ipn` with the UID from the body to get the real status and the canonical UID. Real-world ghost-payment incident: a customer paid but their row stayed pending because the handler trusted the sparse body instead of verifying. PayPlus does not sign IPN callbacks at all (verified against the official PayPlus PHP SDK source; no signature header exists to validate), so phone-home verify is the only defense.

- **Response shape varies between data and transaction keys** - The `/PaymentPages/ipn` response may put `status_code` and `uid` under `data` OR under `transaction` depending on the payment type. Check both: `root.data ?? root.transaction`. `status_code === "000"` means success.

- **Canonical subscription UID comes from the IPN verify call, not the dashboard** - The dashboard shows a row ID (e.g., `<dashboard-row-id>`) which is NOT the API UID. The IPN verify response (`data.uid` or `transaction.uid`) gives the canonical standing-order UID. Backfilling the wrong ID will break subscription lookups.

- **Per-request refURL_callback + dashboard callback = double-fire** - Having BOTH a dashboard callback URL AND `refURL_callback` in the per-request body causes every charge to fire the callback twice, which downstream double-issues invoices. The fix is exactly one callback source. Rule: either (a) set the callback URL in the PayPlus dashboard only and omit `refURL_callback` from the request body, or (b) pass `refURL_callback` per-request only and leave the dashboard callback empty. Never both. Customer-redirect URLs (`refURL_success`, `refURL_failure`, `refURL_cancel`) are safe to pass per-request in all cases.

- **Dashboard "return method: get" controls customer redirect only, not IPN** - The PayPlus dashboard setting "שיטת החזרת מידע: get" governs how the browser redirects the customer after payment. It has nothing to do with the server-to-server IPN, which is always POST. Shipping a GET webhook handler because of that setting is wrong.

- **Middleware can silently 404 your webhook path** - On a split-subdomain app, Next.js middleware was rewriting `/api/payplus/webhook` to a prefixed path, returning 404. PayPlus retries silently failed. Fix: add `/api/*` to the middleware passthrough list. Check your host-split or rewrite rules whenever a webhook stops firing.

- **Hosted Fields may not be enabled** - If `generateLink` returns `hosted_fields_uuid: null`, Hosted Fields are not active on that payment page. Use the hosted-page URL in an iframe instead. Iframe config: 640px height, `allow="payment"`. The postMessage round-trip works because PayPlus redirects back to your domain on completion (same-origin, so `window.top` receives it). After several embed iterations the final working pattern is: iframe of the hosted page with per-session `refURL_callback`, `target="_top"` for the customer redirect, and the dashboard redirect method set to GET to avoid POST/303 browser method-preservation issues.

- **Always add a manual mark-paid escape hatch** - Webhook delivery can fail even with correct config. Add an admin action (server action + button) to mark payment as set up manually. This lets non-engineers resolve stragglers without engineering involvement.

- **IPN fail-closed posture** - If the phone-home verify call to `/PaymentPages/ipn` fails (network error or unparseable response), return 503 and let PayPlus retry. Do NOT fall back to trusting the unsigned IPN body (`statusCode === "000"` from body alone). Retry + idempotency dedupes repeat calls safely.

- **Phone number must be `972XXXXXXXXX` (digits only, no `+`)** - PayPlus rejects phone numbers with a leading `+` or other formatting. Pass raw digits only: `972` prefix followed by the 9-digit local number with no separators or plus sign.

- **Dual-layer duplicate-invoice guard** - A robust guard before issuing any downstream invoice from an IPN uses two layers: (1) a blob/cache keyed on transaction UID as the fast layer, fail-closed; (2) an invoice-provider pre-check by tx UID (e.g. list recent documents over the last 2 days, scan the comments field) before issuing. Both layers must pass; fail-closed when both return unknown (miss on both = do not issue).

- **Company billing: route the invoice to the company tax ID, not the individual payer** - When the PayPlus form includes a company name and a 9-digit Israeli company ID (ח.פ), the downstream invoice must target the company `id_number` field rather than the individual payer. Issuing to the wrong entity is a tax compliance problem.

- **One payment page can back multiple funnels - but charge method is page-level** - `amount` is sent per-request, so a single PayPlus payment-page UID can serve several funnels at different prices (e.g. a base price plus multiple seasonal prices all sharing one page). BUT the charge method (one-time vs recurring standing order) is fixed on the page itself, not per-request, so funnels that must differ on charge method (e.g. recurring subscription vs one-time purchase) need separate payment pages, not one shared page.

- **Gate record-creation behind the verified IPN, not form submit** - Writing customer/contract rows (and sending confirmation emails) at form-submit time means abandon-at-payment leaves orphan rows in the approvals queue and emails a non-paying lead. Create rows in a hidden "awaiting payment" state on submit, then flip them to visible + send emails only from the IPN webhook (make it idempotent by keying off the hidden status), with the manual mark-paid button as the escape hatch if the webhook never fires.

- **Refunds via API: `POST /Transactions/RefundByTransactionUID`** - takes `transaction_uid` (the canonical UID from the IPN verify) + `amount` (partial refunds allowed, up to the original charge). The JSON-stringified `Authorization` header works on this endpoint too (the docs show separate `api-key`/`secret-key` headers; both are accepted). Success = `results.status: "success"`, and the response carries the refund's own new transaction UID + number. `initial_invoice` only works with PayPlus Invoice+ - accounts on an external invoicing provider must issue the credit note (Israeli חשבונית מס זיכוי) themselves. Verified end to end on real refunds.

- **403 error 1010 = IP allowlist, on ALL endpoints, even from a previously working machine** - PayPlus rejects requests from non-allowlisted IPs with `403 {error code: 1010}` regardless of endpoint (`/Transactions/*` AND `/PaymentPages/ipn` both). Credentials are irrelevant; the same request succeeds from allowlisted egress. Workaround when the dashboard allowlist can't be touched: run the call from infrastructure that is already proven to reach PayPlus in prod (e.g. deploy a temporary secret-protected serverless function on the site whose IPN phone-home already works, invoke it, then delete + redeploy). An office IP got 1010 on both endpoints while the site's own egress worked first try.

## Conclusions / best practices

- Auth header: always `JSON.stringify({ api_key, secret_key })`.
- After every IPN POST: call `/PaymentPages/ipn` to get the real status and canonical UID before updating your DB.
- Choose one callback delivery channel (dashboard or per-request `refURL_callback`), not both.
- Store the UID returned by the IPN verify call, not the dashboard row ID.
- Ensure your web framework or middleware does not rewrite or intercept `/api/*` paths before the webhook handler.
- Build a manual mark-paid admin button from day one.

## Doc log

- **2026-08-01** - Added two refund scars: `RefundByTransactionUID` (works with the JSON auth header, partial refunds, external-invoicer credit note stays on you) and 403/1010 = IP allowlist on all endpoints (temporary-serverless-function workaround). Bumped Last validated.
- **2026-06-26** - Added two scars: one page can back multiple funnels via per-request `amount` (but charge method is page-level), and gate record-creation/emails behind the verified IPN to avoid abandoned-cart orphans.
- **2026-06-25** - Added IPN-unsigned confirmation (PHP SDK verified), Hosted Fields final iframe pattern, phone format rule, dual-layer duplicate guard, and company billing routing.
- **2026-06-25** - Initial consolidation of PayPlus integration lessons: auth header format, response-shape handling, IPN hardening, and fail-closed posture.
