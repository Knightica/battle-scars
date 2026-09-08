# Cal.com

**Use for:** Booking/scheduling with real-time webhooks into a CRM funnel (booking created / rescheduled / cancelled), plus click-to-book popup embeds on a website.
**Status:** Active

## Setup & access

- **API v2:** base `https://api.cal.com/v2`, auth `Authorization: Bearer cal_...` (a Cal API key, kept in an env var such as `CALCOM_TOKEN`). Personal API keys: Settings -> Developer -> API keys.
- **Webhooks** drive the real-time funnel. Create either in the dashboard (Settings -> Developer -> Webhooks -> Add) or via `POST /v2/webhooks`. Fields: Subscriber URL, event triggers, optional Secret (HMAC), optional custom payload template, Ping test.
- **Embed (popup):** element-click via `@calcom/embed`. Put `data-cal-namespace="<ns>"` + `data-cal-link="team/<slug>/<event>"` on each "Book a call" button, `href="#"`, and init the namespace once in a small embed script. No page navigation.
- **Trigger names** (exact, note double-L): `BOOKING_CREATED`, `BOOKING_RESCHEDULED`, `BOOKING_CANCELLED`.
- **MCP:** hosted at `https://mcp.cal.com/mcp`, OAuth 2.1, ~34 tools. If your MCP client config needs an explicit transport type field (e.g. `"type": "http"`) alongside the URL, omitting it can make the loader skip the entry silently, so the server never shows up as available at all.

## Scars & gotchas

- **The Cal.com MCP does NOT manage webhooks and is not real-time.** `mcp.cal.com/mcp` covers bookings/event-types/availability but has no webhook CRUD, and polling it is not event-driven. For a live booking -> CRM funnel you MUST use a webhook + your own endpoint; the MCP won't do it.

- **API v1 is decommissioned** - `https://api.cal.com/v1/...` returns `410 { "message": "API v1 has been decommissioned. Please migrate to API v2" }`. Old v1-era keys and any `?apiKey=` query-string auth are dead; v2 uses a bearer header only.

- **Cal's API returns Cloudflare error 1010 to non-browser clients** - `curl`/`urllib` with only an `Authorization` header can get `403 error code: 1010`, which looks exactly like an auth failure and sends you chasing a revoked key. It is Cloudflare rejecting the client signature, not Cal rejecting the token. Send a normal desktop `User-Agent` and the request goes through (then you get the *real* status).

- **`/v2/webhooks` needs NO `cal-api-version` header.** The `cal-api-version` header applies to bookings/slots endpoints; sending it (or omitting it) to `/v2/webhooks` doesn't matter, auth is `Authorization: Bearer cal_...` only. Create: `POST /v2/webhooks` body `{ subscriberUrl, triggers: ["BOOKING_CREATED","BOOKING_RESCHEDULED","BOOKING_CANCELLED"], active: true, secret? }` -> `201`, webhook id at `data.id`. List first (`GET /v2/webhooks`) for idempotency.

- **Webhook payload is wrapped; attendee/booking fields are nested.** Body is `{ triggerEvent, createdAt, payload: {...} }`. Inside `payload`: attendee at `payload.attendees[0]` (`name`, `email`, `phoneNumber`, `timeZone`); event at `payload.type` / `payload.eventTitle`; times `payload.startTime` / `payload.endTime` (ISO); meeting link `payload.videoCallData?.url || payload.location`; booking id `payload.uid`. Parse defensively, phone is only present if the form collected it.

- **The booker's notes are NOT at `payload.notes`** - Cal's notes box (what the person types while scheduling) arrives as the built-in `notes` booking question inside `responses`, with `additionalNotes` / `description` as older aliases. A receiver reading `payload.notes` gets nothing and silently drops the only free text the booker wrote.

- **The embed's `metadata` is NOT forwarded to the webhook** - Stuffing values into `data-cal-config`'s `metadata` object (e.g. an analytics client id) looks like the natural way to carry attribution through a booking, and Cal's *API* docs do describe a booking `metadata` field. But a live booking can arrive at the webhook with no metadata at all: no site marker, no client id, so a conversion gets filed against the wrong source and can't be stitched to the visitor's session. The docs never state this either way. Carry values as **hidden booking questions** on the event type instead (below).

- **Hidden booking questions are the way to pass data through a booking** - Event type -> Advanced -> Booking questions -> Add. The **Identifier** is what matters: it becomes both the URL prefill parameter and the key in the webhook's `responses`, and it is case-sensitive. Make them **optional** (a required hidden field blocks any booking that arrives without a prefill, e.g. from a plain cal.com link) and toggle them hidden **after** adding, or they show on the public booker. "Disable input if the URL identifier is prefilled" is cosmetic once hidden.

- **Prefill must go on the LINK, not in the embed config** - Having added hidden booking questions, setting them as keys on `data-cal-config` (e.g. `{"site":"il","clientId":"..."}`) can still produce EMPTY answers in the webhook `responses`: the fields exist and the identifiers match, but nothing populates them. What works is the query string on the link itself, e.g. `data-cal-link="team/<slug>/<event>?site=il&clientId=..."`, exactly as Cal's own Identifier hint says ("Passed as a URL parameter and can be used to prefill booking questions"). Rebuild the query string from parsed pairs so re-running the stamping code cannot duplicate params.

- **Empty string vs "[object Object]" is the diagnostic** - When a prefilled question comes back as garbage, your reader is wrong. When it comes back EMPTY, the reader is right and the prefill never happened. That single signal separates "fix the parser" from "fix the prefill" and saves a whole deploy cycle of guessing.

- **`responses` shape is not reliably `{ label, value, isHidden }`** - The documented shape is a `{ label, value, isHidden }` wrapper, and for some field types `value` is itself an object (`location` -> `{ optionValue, value }`) or an array (`guests`). A prefilled custom text field can come back as an object that a naive `'value' in v` unwrap renders as `"[object Object]"`. Write the reader to dig for the first string leaf (bare string -> `value` -> `value.value` -> array element), skipping `label` / `isHidden` / `type`, and to return empty rather than garbage. Match identifiers case-insensitively while you are at it.

- **One webhook serves all sites, the endpoint's domain tells you nothing about origin** - Cal posts every booking to a single subscriber URL, so a booking started on one site can arrive at the same endpoint as bookings from any other site sharing that webhook. Do not infer the originating site from which function received the call; carry it in a booking question. A second webhook per site is the wrong fix: it just delivers every booking twice.

- **Pointing the webhook at a branch-preview URL silently kills the integration** - A production site's webhook left pointed at a preview/staging deploy URL for even a few weeks means the downstream conversion event has **never once fired** in production, which reads as "nobody books" rather than "nothing is wired". Always point the subscriber URL at the production hostname, and treat a conversion with zero lifetime events as a wiring bug until proven otherwise.

- **Signature is `x-cal-signature-256` = HMAC-SHA256 over the RAW body.** Verify with the webhook's Secret, constant-time, against the exact raw request bytes (not the re-serialized JSON). The header is only present if a Secret was set on the webhook; if you leave the Secret empty, there's nothing to verify (accept-and-process, harden later). The same secret must live on both the Cal webhook and your receiver's env for verification to run.

- **Signature verification: set the secret in Cal first, then in the app** - When the receiver's webhook-secret env var is unset it should skip verification, so the safe order is Cal -> app. Reversed, the receiver rejects every booking as `bad_signature` until Cal catches up. After setting it, confirm enforcement with an unsigned POST: it must return 401 where it previously returned 200. Note serverless functions only pick up a new env var on a fresh deploy.

- **A "Ping test" (and any event you don't handle) still POSTs to your endpoint.** The dashboard Ping sends a `PING`-type event; unknown/unsubscribed events arrive too if selected. Your handler should 200-acknowledge and ignore anything that isn't a booking event, so a ping/other event never errors or creates CRM noise. Also: the Ping test fails (404) until your endpoint is actually deployed, deploy the function first, then ping.

- **The default booking form's Phone field defaults to +1 (US) and can be made mandatory.** If your audience is outside the US, make phone mandatory (so downstream CRM/messaging has it) and expect the country default to be US unless the attendee changes it. Normalize server-side, don't trust the country selector.

## Conclusions / best practices

- Funnel shape that works: Cal webhook -> thin serverless function (verify HMAC if secret set, parse the wrapped payload) -> fire a background task (e.g. Trigger.dev) that does the CRM writes. Keep the function dumb; the task owns retries.
- A booking that must reach a CRM should be processed server-side from the webhook, never from the embed's success callback: the iframe can be closed before any client-side code runs.
- Treat the webhook payload as the contract, not the docs: log the raw body once when wiring a new field, rather than inferring the shape.
- Subscribe to only the events you handle (`BOOKING_CREATED`/`RESCHEDULED`/`CANCELLED`), not "all", and let any unknown `triggerEvent` return 200 and be ignored, so a ping or future event type never looks like a failure.
- Idempotency: dedupe on `payload.uid` + `triggerEvent`. Cal redelivers, and a booking uid is stable and unique, so it needs no time bucket.
- Point the Subscriber URL at the environment that actually runs the funnel (a preview/branch URL while building; swap to the production domain on promote).
- Treat cancellations as no-downgrade: comment + alert, leave the human to decide status.
