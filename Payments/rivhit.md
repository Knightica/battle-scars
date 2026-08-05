# Rivhit

**Use for:** Israeli invoice generation (חשבונית מס קבלה and credit notes); automated post-payment receipts triggered by PayPlus IPN
**Status:** Active
**Last validated:** 2026-07-06

## Setup & access

- Document creation: `POST /Document.New` with bearer auth.
- Document listing (for idempotency pre-check): `GET /Document.List` filtered by date range; scan `comments` field for the transaction UID.
- Env var `RIVHIT_DOCUMENT_TYPE_INVOICE_RECEIPT` - see document-type gotcha below before trusting this value.

## Scars & gotchas

- **Allocation number (מספר הקצאה) is driven by the customer `id_number` on `Document.New`** - Israel's חשבונית ישראל reform requires a Tax Authority allocation number on tax invoices above a phased amount threshold (pre-VAT: ~₪25k in 2024, ₪20k 2025, dropping toward ₪5k by 2028). Rivhit requests and stamps the allocation number **automatically** for qualifying invoices - the ONE thing the integration must supply is the customer's ת.ז. in the `id_number` field of the `Document.New` body (typed int in the docs; send the 9-char string to preserve leading zeros). No dedicated allocation-number field exists in the request/response schema. If the ID isn't passed, no allocation number can be requested and the invoice is non-compliant above the threshold. Prerequisite: the Rivhit account must be enabled for the SHAAM allocation-number integration (account-level setting, not an API flag). Field confirmed against the live Document.New API reference.

- **Double-invoice incident: multiple duplicate invoices from one real charge** - Root cause: PayPlus fired the IPN twice (dual-callback config bug, see payplus.md) and `Document.New` was not idempotency-guarded. Several duplicate Rivhit invoices were created from a single real charge. Fix: tri-state blob read (hit|miss|error) keyed on transaction UID as the fast layer, plus a `Document.List` pre-check by tx UID covering the last 2 days (scan `comments` field). Both layers must pass; fail-closed when both return unknown (miss on both = do not issue).

- **Env var `RIVHIT_DOCUMENT_TYPE_INVOICE_RECEIPT=3` is MISLEADING** - In one environment this variable was set to `3`, but the actual paid-invoice document type is `2` (חשבונית מס קבלה). Type `3` in Rivhit is חשבונית מס זיכוי (credit note). Trusting the env var name without verifying against the Rivhit account docs will cause paid invoices to be issued as credit notes. Verify the real document type per account before wiring anything up.

- **Refunds: issue a type-3 credit note with negative amounts - never delete** - Israeli tax law prohibits deleting or editing issued documents. To offset a billing error, issue a new document of type 3 (חשבונית מס זיכוי) with negative amounts matching the original. This produces a valid paper trail without altering the original.

- **Rivhit `users` API returns 405 for all write operations (Arbox integration context)** - When integrating Rivhit with Arbox user data, any PATCH / PUT / POST to the Rivhit `users` endpoint returns HTTP 405. The endpoint is read-only; user mutations must go through the Arbox side or the Rivhit dashboard directly.

- **PayPlus double-callback is the primary trigger for duplicate invoices** - Any system that calls `Document.New` from a PayPlus IPN must treat idempotency as non-optional. The single-callback-source rule (dashboard URL or per-request `refURL_callback`, never both) is the upstream prevention; the blob + Document.List guard is the downstream catch. See payplus.md for the full analysis.

## Conclusions / best practices

- Always idempotency-guard `Document.New` with at minimum a blob/cache layer keyed on transaction UID. Add a `Document.List` pre-check as a second layer; fail-closed when both are unknown.
- Verify the real document type for the specific Rivhit account - do not trust env var names alone.
- Offsets and refunds use type-3 credit notes with negative amounts. Never attempt to delete or edit issued documents.
- The primary integration risk is PayPlus callback misconfiguration, not Rivhit itself. Follow the single-callback-source rule in payplus.md.
- For Arbox integrations: user mutations go through Arbox or the Rivhit dashboard; the Rivhit users API is read-only (405 on writes).

## Doc log

- **2026-07-06** - Added the allocation-number (מספר הקצאה) scar: driven by the customer `id_number` on `Document.New`, requested automatically by Rivhit for qualifying invoices. Confirmed the field against the live Document.New API reference; bumped Last validated.
- **2026-06-25** - Enriched with integration history: double-invoice incident detail, document-type gotcha, credit-note refund pattern, Arbox users API 405.
- **2026-06-25** - Initial consolidation from a duplicate-invoice post-mortem.
