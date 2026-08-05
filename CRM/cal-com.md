# Cal.com

**Use for:** Booking/scheduling with real-time webhooks into a CRM funnel (booking created / rescheduled / cancelled), plus click-to-book popup embeds on a website.
**Status:** Active
**Last validated:** 2026-07-24

## Setup & access

- **API v2:** base `https://api.cal.com/v2`, auth `Authorization: Bearer cal_...` (a Cal API key, kept in an env var such as `CALCOM_TOKEN`).
- **Webhooks** drive the real-time funnel. Create either in the dashboard (Settings -> Developer -> Webhooks -> Add) or via `POST /v2/webhooks`. Fields: Subscriber URL, event triggers, optional Secret (HMAC), Ping test.
- **Embed (popup):** element-click via `@calcom/embed`. Put `data-cal-namespace="<ns>"` + `data-cal-link="team/<slug>/<event>"` on each "Book a call" button, `href="#"`, and init the namespace once in a small embed script. No page navigation.
- **Trigger names** (exact, note double-L): `BOOKING_CREATED`, `BOOKING_RESCHEDULED`, `BOOKING_CANCELLED`.

## Scars & gotchas

- **The Cal.com MCP does NOT manage webhooks and is not real-time.** `mcp.cal.com/mcp` (OAuth 2.1, ~34 tools) covers bookings/event-types/availability but has no webhook CRUD, and polling it is not event-driven. For a live booking -> CRM funnel you MUST use a webhook + your own endpoint; the MCP won't do it.

- **`/v2/webhooks` needs NO `cal-api-version` header.** The `cal-api-version` header applies to bookings/slots endpoints; sending it (or omitting it) to `/v2/webhooks` doesn't matter, auth is `Authorization: Bearer cal_...` only. Create: `POST /v2/webhooks` body `{ subscriberUrl, triggers: ["BOOKING_CREATED","BOOKING_RESCHEDULED","BOOKING_CANCELLED"], active: true, secret? }` -> `201`, webhook id at `data.id`. List first (`GET /v2/webhooks`) for idempotency.

- **Webhook payload is wrapped; attendee/booking fields are nested.** Body is `{ triggerEvent, createdAt, payload: {...} }`. Inside `payload`: attendee at `payload.attendees[0]` (`name`, `email`, `phoneNumber`, `timeZone`); event at `payload.type` / `payload.eventTitle`; times `payload.startTime` / `payload.endTime` (ISO); meeting link `payload.videoCallData?.url || payload.location`; booking id `payload.uid`. Parse defensively, phone is only present if the form collected it.

- **Signature is `x-cal-signature-256` = HMAC-SHA256 over the RAW body.** Verify with the webhook's Secret, constant-time, against the exact raw request bytes (not the re-serialized JSON). The header is only present if a Secret was set on the webhook; if you leave the Secret empty, there's nothing to verify (accept-and-process, harden later). The same secret must live on both the Cal webhook and your receiver's env for verification to run.

- **A "Ping test" (and any event you don't handle) still POSTs to your endpoint.** The dashboard Ping sends a `PING`-type event; unknown/unsubscribed events arrive too if selected. Your handler should 200-acknowledge and ignore anything that isn't a booking event, so a ping/other event never errors or creates CRM noise. Also: the Ping test fails (404) until your endpoint is actually deployed, deploy the function first, then ping.

- **The default booking form's Phone field defaults to +1 (US) and can be made mandatory.** If your audience is outside the US, make phone mandatory (so downstream CRM/messaging has it) and expect the country default to be US unless the attendee changes it. Normalize server-side, don't trust the country selector.

## Conclusions / best practices

- Funnel shape that works: Cal webhook -> thin serverless function (verify HMAC if secret set, parse the wrapped payload) -> fire a background task (e.g. Trigger.dev) that does the CRM writes. Keep the function dumb; the task owns retries.
- Subscribe to only the events you handle (`BOOKING_CREATED`/`RESCHEDULED`/`CANCELLED`), not "all", to avoid endpoint noise.
- Point the Subscriber URL at the environment that actually runs the funnel (a preview/branch URL while building; swap to the production domain on promote).
- Treat cancellations as no-downgrade: comment + alert, leave the human to decide status.

## Doc log

- **2026-07-24** - Created from an intro-booking funnel build: MCP-has-no-webhooks, `/v2/webhooks` needs no `cal-api-version` header, wrapped payload field map, `x-cal-signature-256` HMAC over raw body, ping/unknown-events reach the endpoint (200-ignore), phone-mandatory intro form defaults to +1.
