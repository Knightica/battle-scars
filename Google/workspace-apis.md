# Google Workspace APIs

**Use for:** Drive file management, Gmail drafting/sending, Calendar event CRUD, and Sheets read/write - via a service account or OAuth, depending on who owns the data.
**Status:** Active

## Setup & access

- **Consumer OAuth (personal/client-owned Drive):** use the account owner's own OAuth with the narrowest scope that works (`drive.file` when the app only touches files it creates). The automation then owns the folders it creates.
- **Domain-wide delegation (Workspace-owned data):** a service account with DWD acts as a chosen Workspace user. Every scope the SA requests must be pre-authorized in the Admin Console first (see scars). Match the requested scope set exactly to what admin authorized.
- Common scope facts:
  - **Drive (create/move):** full `https://www.googleapis.com/auth/drive`. Share the target folder with the acting user as Editor.
  - **Gmail send/draft:** `gmail.modify` (a superset of `gmail.send`; see scar).
  - **Calendar read/write:** `calendar`.
  - **Calendar delete:** `calendar.events` (`calendar` alone is not enough).

## Scars & gotchas

- **Unverified consumer OAuth app + sensitive/restricted scope: use Testing mode + test user, NOT verification** - An app needing a new sensitive scope (e.g. `calendar.events`) can keep re-consenting with only its old scope. Google's granular consent screen silently **drops any scope the user doesn't tick**, and a new sensitive scope on an app that isn't Testing/verified for that user is refused. The fix is NOT to submit for verification: if the app uses a **restricted** scope (like full `drive`), production verification requires a third-party **CASA security assessment** (weeks, privacy policy, $$). Correct path for a single-user internal tool: OAuth consent screen -> publishing status **Testing**, add the user to **Test users**, then re-consent and click through "Google hasn't verified this app -> Advanced -> Go to app". Enabling the API alone is not enough; the scope must be granted at consent.

- **Re-consent for a NEW scope can silently DOWNGRADE an existing one** - a reauth script that requests `drive.file` + `calendar.events` will mint a token with `drive.file` even if the live token had full `drive`. Google grants exactly what's requested, and existing broader grants are not preserved across a fresh `prompt=consent`. Always request the union of ALL scopes the app needs, or you break the Drive pipeline while adding Calendar. Verify the granted scopes via `tokeninfo` after every re-consent.

- **`drive.file` scope cannot see files other apps create - even inside folders your app owns** - a pipeline where a third-party app drops PDFs into app-created folders breaks on this: `files.list` with `drive.file` returns only the app's own files, silently missing everything else. Watching a folder for third-party uploads requires the full `drive` scope (restricted; unverified consumer apps pass via the Advanced > Go to app consent path). Minting pattern that keeps secrets out of chat: client id/secret in a secret store, a localhost redirect script exchanges the code and writes the refresh token back to the store, then a store-sourced `syncEnvVars` deploy pushes it to the runtime.

- **DWD authorizes `gmail.modify`, not `gmail.send` separately** - a Gmail send can fail with `unauthorized_client` until the requested scope is changed from `gmail.send` to `gmail.modify` to match the admin allowlist. `gmail.modify` is a superset (draft + send covered), so authorizing it covers sending without a separate `gmail.send` entry.

- **A Workspace SA cannot create or move folders in a consumer-Gmail Drive** - a domain-wide-delegated SA acts as a Workspace user; against a personal/consumer Google Drive it is read/share-only and cannot create or move folders. Fix: use the account owner's own consumer-Gmail OAuth with `drive.file` scope so the automation owns the folders from the start.

- **Sheets: a URL written RAW is not a clickable link, and USER_ENTERED coerces number-like strings** - `values.update`/`values.append` default to `valueInputOption: RAW`, which stores a URL as plain text (not linkified) and a formula as literal text. For a clickable cell write `=HYPERLINK("url","label")` with `valueInputOption: USER_ENTERED`. But USER_ENTERED also coerces number-like strings, so it drops a leading zero from values like a card's last-4 ("0432" -> 432). Fix: write only the hyperlink cell with USER_ENTERED and keep RAW for the value columns (two separate range writes around the same row).

- **Sheets: `batchUpdate deleteDimension` destroys an ARRAYFORMULA anchored in a deleted row** - a spilling `=ARRAYFORMULA(...)` lives in ONE anchor cell (e.g. an Age formula in H2 spilling H2:H). Deleting that row (deleteDimension ROWS) removes the anchor and the whole spill vanishes silently; downstream cells just go blank. Fix: after any row delete on a sheet with anchored arrayformulas, re-write the formula into the new anchor row; or only ever delete rows BELOW the anchor. (Some MCPs have no deleteDimension; it can be done via the raw Sheets API + service-account JWT.)

- **Calendar delete requires `calendar.events` scope, not just `calendar`** - a `deleteEvent()` call fails until OAuth is re-run with the scope bumped to `calendar.events`. `calendar` alone covers read/write but not deletion.

- **Adding a new sensitive OAuth scope silently no-ops until it's on the consent screen AND propagated** - Re-running consent for a newly-needed scope (e.g. `calendar.events`) can keep granting only the *existing* scope set, with no error. Three things are required, in order: (1) enable the API in the GCP project (APIs & Services -> Library), (2) add the scope on the OAuth consent screen (**Google Auth Platform -> Data Access -> Add scopes -> Save** - just requesting it in the auth URL is NOT enough), and (3) **wait for propagation - Google's own warning says "5 minutes to a few hours."** A consent completed minutes after saving the scope serves the stale set and hands back the old grant. Always verify the actual grant with the token-introspection endpoint `https://oauth2.googleapis.com/tokeninfo?access_token=<t>` and check `scope` before building on it. **Verification (the Data Access "submit for review" banner) is NOT required** for the token *owner* to use a sensitive/unverified scope on their own account - click through the unverified-app warning (Advanced -> continue); verification only removes the warning and lifts the outside-user cap.

- **Silent Calendar failure: missing env vars swallowed as a non-fatal JSON parse error** - a task returned `ok` while its service-account key + calendar id env vars were absent from prod, throwing "Unexpected end of JSON input" that was caught non-fatally. The task appeared to succeed. Add a post-run verification step (confirm the expected side effect actually landed) on any task that touches Google services with broad error handling.

- **Drive folder naming must be deterministic and timezone-aware** - compute subfolder names (e.g. `MM/YYYY`) against a fixed timezone (e.g. Asia/Jerusalem). If the name is not deterministic, re-runs create duplicate folders and the "ensure child folder" step is no longer idempotent. Always anchor the timestamp to a fixed timezone before building the name.

- **Gmail signatures: no inline SVG** - Gmail, Outlook, and iOS Mail strip inline SVG. Use a table-based row layout (icon-cell + text-cell) with PNG icons hosted at a stable public URL (e.g., a Netlify site).

- **A delegated SA can drive split Google admin APIs (GA4 + Tag Manager)** - a domain-wide-delegated service account can drive the GA4 Admin API (grant access) AND the Tag Manager API (CRUD tags/triggers + publish). Required scopes: GA4 needs `analytics.manage.users` + `analytics.edit`; Tag Manager needs `tagmanager.edit.containers`, `tagmanager.edit.containerversions`, `tagmanager.publish`.

- **A Workspace org policy can block third-party OAuth for sensitive analytics scopes** - some Workspace orgs block third-party OAuth clients from requesting sensitive analytics scopes, so `gcloud auth` comes back without the requested scopes. A delegated SA bypasses that surface entirely.

- **SA key creation can be blocked by the `iam.disableServiceAccountKeyCreation` org policy** - the org admin must grant itself `roles/orgpolicy.policyAdmin`, then disable-enforce the constraint at project level; key creation works ~90 seconds after enforcement propagates.

- **DWD must be authorized manually in admin.google.com - no API path** - after creating the SA, DWD authorization (Security -> API controls, SA client id + scopes) cannot be automated via any API. It must be done by a human in the Admin Console.

- **One GCP project per org boundary** - keep a separate GCP project per org boundary so Workspace + Cloud + SA + billing stay self-contained for a clean spin-off later.

- **Hebrew email subject encoding via the MCP - mojibake** - sending Hebrew email subjects without encoding produces mojibake. Fix: RFC 2047 encoding plus a `from_name` / sender-name param in the send call.

- **Mail sent via a new sending path lands in spam without SPF/DKIM/DMARC** - mail sent through a service account or any new sending path, with no SPF/DKIM/DMARC on the domain, lands in spam for external recipients. Fix: add SPF (e.g. `v=spf1 include:_spf.google.com ~all`, extend with any other sender you use) plus a DMARC `p=none` monitor record; add DKIM.

- **Secret hygiene: SA key JSON must never surface in logs** - an error handler that echoes the full service-account key JSON to stdout leaks it into the transcript/log. Rule: never source a whole `.env` into a shell; read single values via `grep`/`cut` only.

- **Drive: upload immediately after render, capture `fileId` + `webViewLink`** - auto-create the monthly subfolder if missing, upload the rendered PDF immediately after generation, and capture both `fileId` and `webViewLink` for downstream steps. Reinforces the deterministic-folder scar above.

- **Domain-wide delegation fails the WHOLE token request if ANY single requested scope is unauthorized** - a DWD token request that includes even one scope missing from the Admin-console allowlist fails entirely with `unauthorized_client`. You get zero scopes back, not a partial grant. Requesting `analytics.readonly` (never added to the allowlist) alongside an authorized `analytics.edit` breaks the entire token, and the error names none of the scopes. Fix: request ONLY the scopes actually authorized in the Admin console, never a superset "just in case".

- **`gmail.compose` can create drafts but cannot LIST or APPLY labels - filing a draft under a label needs `gmail.modify`** - `drafts.create` ignores any `labelIds` in the payload, and `labels.list` / `messages.modify` are not authorized under `gmail.compose`. To tag outreach drafts with a label: authorize `gmail.modify`, look up the label id once via `labels.list`, then after `drafts.create` call `users.messages.modify` on the returned `draft.message.id` with `{ addLabelIds: [labelId] }`. Make the label call **best-effort** - the draft already exists, so a labeling failure must not throw and trigger a retry that creates a SECOND draft.

- **Sheets: SUM silently skips text-formatted currency cells** - a hand-typed currency value (e.g. `₪300.00`) is stored as TEXT and renders identically to a numeric cell with currency format, but SUM/SUMIF ignore it, so totals silently under-count. Audit with `=COUNT(range)` vs row count or `ARRAYFORMULA(IF(ISNUMBER(...)))`. Two such cells once hid a chunk of real revenue in a totals row.

- **Sheets: fixed-range totals go stale under values.append** - `append` with `INSERT_ROWS` places new rows BELOW the end of a `=SUM(J2:J65)` range, and a range whose end row is the last data row does NOT auto-extend, so the total quietly freezes. Use self-adjusting totals anchored to the totals row itself: `=SUMIF(K$2:INDIRECT("K"&ROW()-1),"Approve",J$2:INDIRECT("J"&ROW()-1))` - survives inserts and never goes circular.

- **Sheets: renaming a tab breaks every automation pinned to the old name** - values API ranges are by tab TITLE (`'Sheet1'!A:N`), so a human renaming a tab turns every task write into `Unable to parse range`. If automations write to a sheet humans touch, treat tab names as config (env var) and expect them to drift; on 400 parse errors, list actual titles via `spreadsheets.get` before debugging anything else.

## Conclusions / best practices

- Match the DWD scope exactly to what the Workspace admin has authorized. Mismatches surface as `unauthorized_client`, not a helpful 403.
- For any personal/client-owned Drive (consumer account): use the owner's OAuth with `drive.file`, not a Workspace SA.
- Always share Drive parent folders with the acting Workspace user as Editor before expecting the SA to write there.
- Add post-run verification on Calendar/Drive tasks - Google API errors can be caught silently and the task still returns success.
- Compute Drive subfolder names deterministically (include the correct timezone) so idempotent re-runs are a no-op.
- Never use inline SVG in email HTML. Host icon PNGs at a stable public URL and reference them with `<img src="...">`.
- Under domain-wide delegation, `gmail.modify` is a superset of `gmail.send`: a JWT can call `users.messages.send` with only `gmail.modify` authorized, so you need not add `gmail.send` to the scope list.
- A fresh GCP org enforces `iam.disableServiceAccountKeyCreation` by default; SA key creation fails until you grant the admin `roles/orgpolicy.policyAdmin` and run `gcloud resource-manager org-policies disable-enforce iam.disableServiceAccountKeyCreation --project=<proj>`. Enforcement propagation takes ~2 min (several key-create attempts fail first).
