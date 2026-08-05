# Calendly

**Use for:** Meeting scheduling detection, cancellation handling, deduplication against Google Calendar events
**Status:** Active
**Last validated:** 2026-04-19

## Setup & access

- A webhook-free design is viable: all event detection can run via a polling loop against the Calendly API.
- Personal access token for auth; user URI is auto-fetched at startup.
- The same meeting can appear in two places: as a Calendly event (iCalUID ending `@calendly.com`) and as a synced Google Calendar copy (iCalUID ending `@google.com`, organizer is the calendar owner). Both pollers must recognize both shapes.

## Scars & gotchas

- **Dual-event dedup (iCalUID + organizer, three layers required)** - A Calendly booking can surface through two pollers simultaneously: the Calendly poller finds the canonical event (iCalUID `@calendly.com`) and the Google Calendar poller finds the synced copy (iCalUID `@google.com`, but organizer email includes `calendly.com`). Without dedup, both pollers schedule a confirmation email. Three-layer guard: (1) `isCalendlyEvent()` checks `iCalUID.endsWith("@calendly.com")`; (2) fallback checks `organizer.email.includes("calendly.com")` (catches synced copies); (3) `scheduleOnce()` in the job queue skips enqueue if a job of the same `type + lead_id` is already pending, running, or completed within 24h. Layer 1 alone is unreliable when Calendly syncs to the calendar owner's account (iCalUID lands as `@google.com`).

- **Cancellation reason must be captured and noted** - The poller detects cancellations. When one fires, capture the cancellation reason and write it as a note on the CRM deal before transitioning the deal stage. The reason is needed for an AI-generated reschedule email.

- **Manual vs invitee cancellation - different outcomes** - Invitee-initiated cancel: generate a reschedule draft email and move the deal to a Meeting-Cancelled stage. Host-initiated (manual) cancel: delete the Calendly event and the Google Calendar event silently using `sendUpdates: "none"` so the invitee receives no notification. Never auto-send a notification on a manual cancel.

- **No useful outbound webhook** - Calendly's webhook payload may not fit an integration's needs (matching an invitee to a CRM deal can require a lookup the webhook alone cannot drive). In that case all detection is polling-based. If you need event-driven Calendly processing, plan for a polling architecture, not a webhook handler, unless you can guarantee the payload carries everything your CRM lookup needs.

## Conclusions / best practices

- Always apply all three dedup layers: iCalUID suffix check, organizer email check, and a job-queue idempotency guard keyed on `type + lead_id` with a 24h window. Relying on iCalUID alone fails when Calendly syncs the event under the calendar owner's Google identity.
- Cancellation handling must branch on who cancelled. Invitee cancels get a draft reschedule email. Manual (host) cancels get a silent Calendly + GCal delete with `sendUpdates: "none"`.
- Always capture and persist the cancellation reason before acting on it. It drives the personalized reschedule email.
- Design Calendly integrations around polling, not webhooks, unless you can guarantee payload completeness for your CRM lookup needs.

## Doc log

- **2026-06-25** - Initial consolidation.
