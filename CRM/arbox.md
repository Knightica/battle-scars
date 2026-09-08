# Arbox

**Use for:** Gym member + class scheduling backend - member records, class schedule, coach session reports
**Status:** Active

## Setup & access

- Auth via `api-key` header (value kept in an env var such as `ARBOX_API_KEY`). No OAuth, no Bearer token.
- Base URL kept in an env var (e.g. `ARBOX_BASE_URL`). Location/floor IDs are account-specific integers passed as `location_id`.
- The public API is read-only for user/staff mutations. Any record creation or editing must go through the Arbox dashboard manually or via a browser-automation flow (e.g. Puppeteer).
- No staff-create endpoint exists in the public API (`GET /v3/users/allStaffMembers` is read-only; no POST equivalent confirmed as of this writing).

## Scars & gotchas

- **Public API is read-only for user mutations** - PATCH, PUT, and POST on `/v3/users` all return `405 Method Not Allowed`. There is no documented or undiscovered staff-create endpoint either. Fixing bad member records (corrupt phones, duplicates) requires manual Arbox dashboard edits or a browser-automation flow, no programmatic path exists.

- **`++972` phone corruption** - An n8n "Create User" node templated the phone as `"+{{ contact.phone }}"`, which prepended `+` onto an already-`+972...` stored value, producing `++972...` for dozens of records and single-`+`-prefixed records for hundreds more. Fix: normalize phones to digits-only `972XXXXXXXXX` (no `+`, no leading zero) before any write. `.replace('+', '')` patched the source; the existing dirty records in Arbox had to be cleaned manually (no API path). Lesson: never pass `+`-prefixed phone values through template strings that add their own `+`.

- **Pagination via `next_page_url`, filter `status === "active"`** - Both `GET /v3/reports/classesSummaryReport` and `GET /v3/schedule` paginate via `extra.pagination.next_page_url`. Fetch pages in a loop until `next_page_url` is null/absent. Always filter `status === "active"` to exclude cancellations and no-shows from session counts and schedule lookups.

- **`/v3/schedule` is GET; `registration_details` must be integer `1`; registrants are under `registration_Details` (capital D)** - Older flows POSTed to `/v3/schedule` and got `405 Method Not Allowed` (supported: GET/HEAD/PATCH). It is a **GET** with `location_id` + `from_date` + `to_date` (both dates required) as query params. `registration_details` must be the integer **`1`**, the string `"true"` returns `400 "registration details must be a boolean"`. Registrants come back under the key **`registration_Details`** (capital D), each with `full_name`, `phone`, `user_role` (filter out `staffMember`), `checked_in`. The window is capped at ~7 days; for a longer horizon, re-check on a rolling window rather than widening the range.

- **`activeMembershipsReport` is a LIVE snapshot, not a time machine.** Passing a past `fromDate`/`toDate` does not reconstruct that date; memberships cancelled since then have vanished from the report. Historical paying-member counts must be rebuilt from multiple sources: current rows covering the date, plus `canceledMembershipsReport` rows covering it, plus renewal rows proven by that month's successful RECURRING charges in `transactionsReport`.

- **manage.arboxapp.com profile-URL ids are a different id space than public-API `user_id`.** `/v3/users/{id}` exists and works for report user_ids (a 9-10M range), but UI profile ids (a much smaller number) 404 and appear in NO field of any public report or the full `/v3/users` list. There is no API path from a profile URL to a user; ask for name/phone instead.

- **An invalid report name errors back the complete list of valid report names.** `GET /v3/reports/xx` returns a 400 whose message enumerates every `reportName` the account accepts, the fastest way to discover endpoints like `inactiveMembersReport`, `salesReport`, `renewalsReport`. Also: `transactionsReport` silently returns 0 rows for over-wide date ranges, query month by month.

## Conclusions / best practices

- Normalize all phone numbers to digits-only `972XXXXXXXXX` before writing to Arbox or any downstream platform. Never embed `+`-prefixed values in template strings.
- Treat the Arbox public API as read-only for member records. Plan fixes as manual dashboard actions or browser-automation scripts, not API calls.
- Always paginate `classesSummaryReport` and `schedule` endpoints via `next_page_url`, and filter `status === "active"` to get clean session data.
