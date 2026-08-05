# Glossary

Only the terms whose meaning in these notes is project-specific or conventional, the ones you can't infer from general knowledge. Standard acronyms (HMAC, RLS, TTL, DDL, CAPI, GA4, GTM, ANAME, SPF/DKIM/DMARC, CTR, idempotent, mojibake, etc.) are assumed known; look them up normally. If a term isn't here, it means "the usual thing."

- **MER** - Marketing Efficiency Ratio: total store revenue / total ad spend, using the store (e.g. WooCommerce) as the source of truth. We treat this as the real ROI number because platform-reported **ROAS** over-claims (each ad platform counts conversions it can't actually own), so store-reconciled MER is the honest figure.
- **system-user token (Meta)** - the never-expiring token (`expires_at: 0`) issued from a Business portfolio. Convention: one per account, treated like a root key.
- **project ref (`proj_*`)** - a Trigger.dev project ID. Hard rule: one project ref per codebase. Sharing one silently kills the other codebase's scheduled tasks.
- **`syncEnvVars` (allowlist)** - a Trigger.dev build extension that pushes the local `.env` to prod on deploy; scope it to a per-project prefix allowlist so one project's secrets never reach another's prod.
- **soft-skip** - a cron convention: catch a known transient, skip silently with no alert (the job reruns). Webhooks do the opposite: throw, so the source retries.
- **onFailure → alerts bot** - route background-job failures (e.g. the Trigger.dev `onFailure` hook) to a dedicated per-project alerts bot, never a shared conversational bot.
- **fail-closed** - when a safety check (e.g. a duplicate-invoice guard) can't confirm it's safe, refuse the action and let the provider retry, rather than proceed.
- **mirror column (Monday)** - a column reflecting a linked board's value. It returns its value only in `display_value` (via the `MirrorValue` GraphQL fragment), not `.text` / `.value`, a silent-null trap.
- **ח.פ** - Israeli company registration number. Company invoices route to this, not to the individual payer.
