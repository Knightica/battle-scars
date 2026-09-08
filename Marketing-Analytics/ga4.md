# Google Analytics 4 (GA4)

**Use for:** Website analytics, purchase funnel tracking, cross-source attribution (GA4 last-click as the middle layer between store truth and ad-platform claims).
**Status:** Active

## Setup & access

- Create a dedicated service account per property via gcloud. Keep analytics SAs separate from any unrelated automation service accounts.
- Enable the Analytics Admin API and the Analytics Data API on the GCP project.
- Grant Viewer to the SA **via the Admin API**, not the GA4 UI (see scars). Use `v1alpha/properties/{id}/accessBindings` with a token generated in OAuth Playground with the `analytics.manage.users` scope.
- Download the SA JSON key to a stable local path. Hardcode this path in `.mcp.json` (see scar below - do not use `${VAR}` interpolation).
- Use the official Google `analytics-mcp` (installed via pipx). Do not use community alternatives.
- Validate: call `get_account_summaries` - should return the property name and property ID without error.

## Scars & gotchas

- **A GA4 key event (conversion) can be created for an event name GA4 has never seen** - `keyEvents.create` does not require the event to have fired. The Admin UI only offers names it has already recorded, which makes it look like a never-fired event cannot be marked as a conversion, and belief in that cost days of "waiting for it to fire first" before someone tried the API directly. It is a UI limitation, not an API one. `POST https://analyticsadmin.googleapis.com/v1beta/properties/{id}/keyEvents` with `{"eventName": "schedule", "countingMethod": "ONCE_PER_EVENT"}` succeeds against an unseen name, and the first real occurrence is then counted as a conversion from the start rather than being lost to reporting. Auth: a service account with domain-wide delegation and the `analytics.edit` scope, impersonating a user who admins the property. Mark planned conversions at build time, before launch, so the very first one counts.

- **The Realtime API double-reports Measurement Protocol events across minute buckets - never debug duplicates with it** - A single MP `POST /mp/collect` showed as `eventCount: 2` in `runRealtimeReport`, split across two adjacent `minutesAgo` buckets. Proven with a control: one throwaway event name sent exactly once, reported twice. Chasing this cost an hour of hunting for a phantom second sender. The standard (non-realtime) report is authoritative and showed the true count of 1. Use realtime only to confirm an event ARRIVES; use `runReport` to confirm HOW MANY.

- **`/debug/mp/collect` validates a stream's measurement id + MP secret without recording anything** - The best way to prove a new data stream's server-side credentials before any real traffic: same body as `/mp/collect` but returns `validationMessages`. An empty array means the id/secret pair and payload are accepted. Nothing lands in the property, so it costs the stream no junk data - much better than firing a fake conversion.

- **Two data streams in one property do NOT share an identity space** - Splitting an apex domain and its ccTLD or subdomain sibling into separate streams (same property) keeps key events, the Ads link and reporting unified, but a visitor crossing the two domains counts as two users and two sessions, and each domain shows as a referral in the other's stream. Cross-domain stitching needs both domains on ONE measurement id, which defeats the split. Don't promise cross-domain continuity as a benefit of the single-property choice.

- **The Measurement Protocol silently DROPS an event with a malformed `session_id` - and still answers 2xx.** MP has no meaningful success signal on the production endpoint: a bad payload returns the same `204` as a good one, so any "did the POST succeed?" check reports green while the event is discarded. This cost one site every server-side conversion it ever sent. The trigger was Google shipping **two formats** for the `_ga_<STREAM>` session cookie:

  | Format | Value | Delimiter |
  |---|---|---|
  | **GS2** (current) | `GS2.1.s1784965297$o1$g0$t1784965297$j60$l0$h0` | `$`, session id prefixed `s` |
  | GS1 (legacy) | `GS1.1.1699990000.3.1.1699990500.0.0.0` | `.` |

  A parser written for GS1 (`cookie.split('.')[2]`) returns the whole `s1784965297$o1$g0$...` tail on a GS2 cookie. GA4 then drops the event. `generate_lead` and `schedule` never recorded once, while the sending tasks logged `sent: true`.

  **Parse both, and only ever emit bare digits:** `v.match(/(?:^|\.)s(\d+)(?:\$|$)/)` for GS2, falling back to the dot-split for GS1. **Guard server-side too** - refuse to send a non-numeric `session_id`, because an *unstitched* event still counts while a *dropped* one does not. `client_id` (from `_ga` = `GA1.1.<c1>.<c2>` -> `<c1>.<c2>`) was unaffected; only the session cookie changed format.

  **The false positive that hid it for a day:** the one conversion that did land was a manual test whose payload carried no session id at all, so no `session_id` was sent and GA4 accepted it. A smoke test that omits the field under test proves nothing - always verify with a **real** captured cookie value. To validate a payload properly, POST it to `https://www.google-analytics.com/debug/mp/collect` (same params), which returns `validationMessages`; the production endpoint never will.

- **GA4 UI blanket-rejects service-account emails** - You cannot add a service-account email as a user through the GA4 Admin UI. The UI silently fails or shows an error. The only working path is the Admin API (`v1alpha/properties/{id}/accessBindings`) with a manually obtained token.

- **gcloud's stock OAuth client silently strips analytics scopes** - When using gcloud to generate an OAuth token, the `analytics.manage.users` scope gets stripped. The result is a valid-looking token that still fails the Admin API call with a permissions error. Use OAuth Playground directly for the one-time admin grant.

- **`${VAR}` interpolation in `.mcp.json` fails silently when env is not pre-sourced** - If the client is launched without the env loaded, `${VAR}` placeholders in `.mcp.json` reach the MCP process unresolved, causing the tool to fail with "File not found" on the credentials path. Symptom: `get_account_summaries` returns a key-file error even though the file exists. Fix: hardcode the absolute path to the GA4 SA JSON key in `.mcp.json`. The path is not a secret. Alternative: add an `.envrc` (direnv) so every terminal entry auto-sources the env.

- **Revenue mapping differs per account: document it explicitly** - A property's GA4 revenue may map to store net + shipping (not gross, not net alone). This must be documented per property because it affects how GA4 numbers are compared to store books. Normal GA4 order coverage vs the store is around 85-90%.

- **UTM fragmentation from the ad platform fragments GA4 attribution** - An ad platform generating multiple campaign-name variants from one campaign causes GA4 to split that campaign's sessions across multiple rows. Results in apparent lower traffic per source than reality and makes channel reporting unreliable. Fix is upstream at the ad-account level (see `meta-ads.md`).

- **Prefer official `analytics-mcp` over community alternatives** - The official Google `analytics-mcp` (pipx-installed) is the stable choice.

- **A property may live in its own dedicated GCP project** - Different properties can live in different GCP projects, each with its own SA and Viewer grant. Set data retention to the maximum (14 months on free GA4). Always confirm which GCP project houses the property's SA before running Admin API calls or diagnosing access issues.

- **Custom medium `pr` does not auto-appear in GA4 default Channels - shows as "Unassigned"** - GA4's default channel group does not include custom mediums like `pr`. Traffic is captured correctly but falls into "Unassigned" in the Channels report. GA4 does not allow editing the default channel group. Fix: Admin -> Data display -> Channel groups -> create a new group with rule `Session medium matches regex ^(pr|press|earned)$` -> channel name "PR / Earned Media". In reports and explorations you must select this custom channel group as the dimension; the default Channels report keeps showing "Unassigned" regardless. Custom channel groups apply retroactively within the data-retention window, so existing PR sessions backfill automatically.

- **A custom channel group REPLACES the default rather than extending it - clone the default rules or everything lands in "Unassigned"** - The obvious build is a one-rule group for your custom mediums. Do that and the group is useless: it is a self-contained grouping dimension, so every session that does not match a rule (i.e. all your organic, direct, social and email traffic) falls into "Unassigned" *within that group*. The cheap fix is that `GET v1alpha/properties/{id}/channelGroups` returns the **system "Default channel group" with its full rule set** (19 rules as of writing), and each rule is just `eachScopeDefaultChannelGroup EXACT "<name>"`. Read it, prepend your own rule, POST the concatenation - a drop-in replacement of the default report plus your channel, in one call. Rules are evaluated in order, so the custom rule must come first.

- **`channelGroups` filters use `eachScopeMedium`, and the field list is not the Data API's** - Building a custom channel rule, `eachScopeManualMedium` (the name the GA4 UI's "Session manual medium" implies) 400s with `Referenced dimension 'eachScopeManualMedium' does not exist`. The working field for medium is **`eachScopeMedium`**. Also: `stringFilter` here has **no `caseSensitive` field** even though the Data API's does - sending it fails with `Unknown name "caseSensitive"`. Match types that do work: `EXACT`, `FULL_REGEXP`. Auth is the same DWD service account + `analytics.edit` scope used for key events.

- **Lead definition "form submit OR WA click" double-counts leads vs CRM truth** - When a Lead event fires on both form submits and WhatsApp button clicks, the same event feeds the ad platform Pixel/CAPI and inflates ad-platform-reported Leads well above true CRM lead count. For lead-gen campaign evaluation, use CRM lead count as ground truth, not the ad platform's Lead metric.

- **Subdomain cookie sharing removes the cross-domain linker requirement - but verify the tag is present** - All subdomains of a root domain share the root cookie, so GA4 session attribution (including UTM source/medium carried from the landing page) persists through funnel subdomains without a cross-domain linker, provided each funnel subdomain fires the same GA4 stream (`G-XXXXXXX`). Verify the GA4 tag is actually live on each funnel subdomain before trusting PR -> conversion attribution paths; the cookie sharing only helps if the tag is there.

- **UTM convention standard** - Keep consistent UTM tagging across the account: paid social `utm_source=facebook` or `instagram`, `utm_medium=paid_social`; chat marketing `utm_source=manychat`, `utm_medium=<flow-name>`; QR/physical `utm_source=<location-slug>`, `utm_medium=qr`; PR/earned media `utm_source=<publication-slug>`, `utm_medium=pr`, `utm_campaign=<publication>_<yyyy-mm>`. The `pr` medium convention aggregates all earned media into one filterable bucket (see custom channel group scar above).

- **Measurement Protocol secret creation returns 400 FAILED_PRECONDITION until User Data Collection is acknowledged** - `POST .../dataStreams/<id>/measurementProtocolSecrets` fails with `400 FAILED_PRECONDITION` when the property has not accepted the User Data Collection Acknowledgement. The error does not say so. Fix: acknowledge it first via the Admin API (`properties/<id>:acknowledgeUserDataCollection`) or in the GA4 UI (Admin -> Data collection and modification -> Data collection), after which the secret creates cleanly. Required before any server-side MP event pipeline can send at all.

## Conclusions / best practices

- **One SA per property.** Cleaner audit trail; revoke per-property without touching others.
- **Hardcode the SA key path in `.mcp.json`.** File paths are not secrets. Env-var interpolation in `.mcp.json` is a known failure mode.
- **Always document the GA4-to-store revenue mapping** at the top of the project docs (e.g., "GA4 revenue = store net + shipping").
- **GA4 is the middle-layer source of truth** in the three-source stack (store = ground truth, GA4 = last-click, ad platform = platform claim). Normal GA4/store order coverage: 85-90%.
- **UTM must be clean upstream (ad-account level) before GA4 attribution is trustworthy.** Don't spend time debugging GA4 channel grouping until the UTM source is fixed.
- **Use MER, not GA4 revenue, for client-facing ROI.** GA4 is useful for funnel and traffic analysis; the cash number comes from the store.
