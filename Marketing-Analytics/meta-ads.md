# Meta Ads (Marketing API)

**Use for:** Paid social campaign management, lead generation, audience building, ad performance reporting via the Meta Marketing API (Graph API v21.0+).
**Status:** Active

**TL;DR:** Build a **Business-type** app + one never-expiring **system-user token per account**; hit the Graph API directly (no MCP). Two ways this loses money: (1) a `targeting` POST is a **full replacement** - GET the whole object, change one field, POST it all back, or you silently wipe audiences; (2) report client ROI as **MER** (store revenue / ad spend), never platform ROAS, which over-claims several-fold. Also: exclude Audience Network from Sales campaigns, and fix UTMs at the **account level**, not per-campaign.

## Setup & access

- Create a **Business-type** app inside the Business portfolio. Use the legacy "Other" path in the app-creation flow. Consumer-type apps cannot access the Marketing API (see scars).
- Add the Marketing API product to the app.
- Create a system user inside the Business with full-control permissions on: ad account, Page, IG, Pixel, Catalog.
- Generate a never-expiring system-user token from inside the Business portfolio. Validate with `debug_token` - look for `type: SYSTEM_USER` and `expires_at: 0` plus all required scopes.
- Store in `.env` as `META_APP_ID`, `META_BUSINESS_ID`, `META_AD_ACCOUNT_ID`, `META_ACCESS_TOKEN`. Never commit.
- No official Meta Ads MCP yet. Hit the Graph API directly:
  ```bash
  set -a; source .env; set +a
  curl -sG "https://graph.facebook.com/v21.0/$META_AD_ACCOUNT_ID/insights" \
    --data-urlencode "fields=spend,impressions,clicks,actions,action_values,purchase_roas" \
    --data-urlencode "date_preset=last_30d" \
    --data-urlencode "access_token=$META_ACCESS_TOKEN"
  ```
- One system user per account for a clean audit trail and easy per-account revocation.

## Scars & gotchas

- **Consumer-type app is a dead end** - Meta blocks Marketing API access for Consumer-type apps entirely. Rebuild as a Business-type app via the "Other" (legacy) path inside the Business portfolio.

- **Business-admin system user sees ALL portfolio ad accounts including inactive ones** - Not a bug; expected behavior when the system user holds Business admin. Legacy inactive accounts show up alongside the active one. Just FYI, not a problem.

- **Audience Network (AN) delivers near-zero-converting impressions** - AN placements in Sales-goal campaigns burn budget on impressions that don't convert. Fix: create Sales campaigns excluding Audience Network placement, or measure AN separately on a reach KPI only. Observed: hundreds of AN sessions with zero purchase attribution in a month.

- **Creative fatigue signal: CTR halving** - When CTR drops more than around 40% while impressions are maintained (impressions held or growing), the creative is fatigued. Flag for refresh.

- **Meta Instant Forms extract only the fields defined in the form** - A form defining exactly `full_name`, `email`, `phone_number` means any pipeline reading `/{form_id}/leads` gets only those columns.

- **API rate limit error #17 ("User request limit reached") persists 10-60+ min** - Sliding-window limit. A burst of diagnostic queries in one session can lock out further reads for over an hour. Batch reads, avoid repeated `/{ad_account_id}/ads` listings in the same session.

- **`url_tags` field quirk on single-ad GETs and batched-id reads** - The field is queryable on `/{ad_account_id}/ads` list calls, but throws `(#100) Tried accessing nonexisting field` on single-ad GETs (`/{ad_id}`) and batched-ids reads (`/?ids=...`). Writes via POST still work fine. Always use the listing endpoint to verify url_tags after writing.

- **UTM: static values only, no Meta dynamic placeholders** - `{{site_source_name}}` and similar Meta dynamic vars are fragile across placement variants and break GA4 attribution grouping. Use static `utm_source=facebook`. Add placement granularity with a separate `placement` param. The proven template:
  ```
  utm_source=facebook&utm_medium=paid_social&utm_campaign={{campaign.name}}&utm_content={{ad.name}}&utm_term={{adset.name}}&utm_id={{ad.id}}&placement={{placement}}
  ```

- **Multi-variant UTM fragmentation breaks GA4 attribution** - Meta can generate several campaign-name variants from a single campaign ("Sales | X", "X", "Sales+|+X", literal `{{campaign.name}}` placeholder), splitting GA4 sessions across multiple rows instead of one. Fix at the Meta **account level**: Ads Manager -> Account-level URL parameters. This is the canonical fix, not per-campaign.

- **Meta over-claims purchases several-fold vs GA4 last-click** - Meta's attribution window (view-through + click) inflates purchase counts relative to GA4 last-click. Report client ROI as MER (store sales / ad spend, using the store as source of truth), not platform ROAS.

- **Campaign-type mismatch: evaluate each campaign on its own KPI** - A Traffic/OUTCOME_TRAFFIC campaign with an IG-profile destination should be evaluated on cost-per-profile-visit and follower gain, not ROAS. Applying ROAS to it will always look like a loss.

- **`targeting` is a full-replacement document, not a merge target** - POST `targeting={...}` to `/{adset_id}` replaces the entire targeting object; anything omitted is deleted silently. Example: POSTing `{publisher_platforms: [...]}` alone wipes geo, age, custom audiences, and everything else. Error when geo is wiped: `"error_user_title": "Location is missing" / "error_user_msg": "Add at least one location or choose a custom audience."` Rule: always GET full targeting with explicit field expansion, modify only what you need, POST the complete object back.

- **Advantage+ Audience locks edits to custom_audiences/interests - error subcode 1359202** - When an ad set has `targeting_automation.advantage_audience: 1`, any PATCH that "changes" `custom_audiences`, lookalike, or detailed targeting is rejected, even if you re-POST the exact values you just fetched. Root cause: GET responses include `[{id, name}]` for custom_audiences and interests, but POST expects `[{id}]` only. The shape difference makes Meta read it as a modification. Error: `"error_subcode": 1359202, "error_user_title": "Cannot Disable Advantage Options", "error_user_msg": "While using Advantage+ audience, the following settings can't be changed: Advantage Detailed Targeting, Advantage Lookalike and Advantage Custom Audience."` Workarounds: (a) strip `name` from `custom_audiences` and `flexible_spec.interests` before POSTing, send `[{id: "..."}]` only; (b) if still rejected, toggle `advantage_audience: 0`, apply the change, toggle back to `1` - two POSTs. Do NOT strip the field from the payload hoping to leave it alone; absence sets it to empty (see scar below).

- **Absent field = empty array: stripping custom_audiences or flexible_spec deletes them** - If `custom_audiences` or `flexible_spec` is absent from a POST payload, Meta sets it to `[]` and silently deletes all audiences and interests. This can nuke a lookalike, a retention custom audience, and an ad set's entire interest set in a single API call. Recovery via Meta activity log: `GET /{adset_id}/activities?fields=event_type,event_time,extra_data,object_name` - the `extra_data` field contains the full prior state as a human-readable diff (audience names, interest names, locations, age, placements); use it to reconstruct the targeting spec.

- **age_max < 65 is incompatible with Advantage+ Audience via API - error subcode 1870189** - When `targeting_automation.advantage_audience: 1`, the API enforces `age_max >= 65`. Any attempt to set a lower value fails with: `"error_subcode": 1870189, "error_user_title": "Maximum age is below threshold", "error_user_msg": "With ad sets that use Advantage+ audience, the maximum age audience control can't be set to lower than 65. You can add a lower maximum age as a suggestion instead when creating or editing an ad set in Ads Manager."` There is no API field for "suggested age max" - that control exists only in the Ads Manager UI. Options: accept hard age 18-65 with Advantage+ on (Meta extends audience by age), or turn Advantage+ off so a hard age_max < 65 can be enforced.

- **Safe-edit checklist for any ad set targeting PATCH** - Before any targeting update via the API: (1) GET full targeting with explicit field expansion - `?fields=targeting{publisher_platforms,custom_audiences,flexible_spec,targeting_automation,age_min,age_max,geo_locations,genders,brand_safety_content_filter_levels}`; (2) strip `name` from all nested ID objects - POST shape must be `[{id: "..."}]` only; (3) modify the smallest set of fields that achieve the goal; (4) POST back the full targeting object, not a partial; (5) verify via a second GET with the same field expansion; (6) pause the ad set first if testing - the activity log is the rollback path but live spend on broken targeting is expensive.

- **Account-level placement controls are cleaner than per-ad-set publisher_platforms patches for AN exclusion** - To exclude Audience Network across an entire ad account without touching each ad set's targeting payload: Ads Manager -> ad set editor -> Placements section -> "Placement controls for this ad account" -> toggle "My business can only advertise on specific placements" -> uncheck Audience Network. Despite the UI path living inside the ad set editor, the scope is account-wide and propagates to all campaigns automatically; no targeting-payload surgery required per ad set.

- **CAPI server-side: SHA-256 all PII, use stable event_id for browser/server dedup** - For Lead (or other) events fired server-side via the Conversions API, all PII fields (email, phone) must be SHA-256 hashed before sending. Use a stable `event_id` shared between the browser pixel event and the server-side CAPI call so Meta can deduplicate the pair and not double-count. A common pattern fires Lead events server-side from a background task on CRM Lead creation.

- **Pixel dedup debugging: /tr and gaawe pings may be invisible in DevTools even in a real browser** - When troubleshooting browser pixel + GA4 Lead event deduplication, the expected network calls (`/tr` for Meta Pixel, `gaawe` for GA4) may not appear in DevTools, even in a non-headless real browser. Hypothesis: the pixel is in CAPI-only / server-side-only mode with no client-side call to intercept. Verification path: Meta Events Manager -> Test Events tab, which shows both browser and server events in real time independently of DevTools.

- **Migrating off all-in-one pixel plugins** - Moving from an all-in-one pixel plugin (e.g., PixelYourSite) to GTM-managed Meta Pixel and GA4, plus Meta CAPI server-side (via a server webhook or workflow proxy), improves iOS/Safari attribution without third-party cookie dependency.

## Conclusions / best practices

- **Business app + system user per account.** One setup per portfolio. Never reuse across accounts.
- **Never-expiring tokens come only from the Business portfolio + app path.** OAuth user tokens expire. System-user tokens do not (`expires_at: 0`). Secure them like root API keys - they control every ad account under the Business.
- **Client-facing metric = MER (store revenue / ad spend).** Platform ROAS stays internal. MER uses store gross as numerator.
- **Evaluate each campaign on its designed KPI.** Sales campaigns: ROAS/MER. Traffic/Brand campaigns: CPV, followers, reach. Lead-gen campaigns: CPL.
- **Fix UTM at the Meta account level,** not per-campaign. Account-level URL parameters in Ads Manager override per-campaign settings and prevent fragmentation drift.
- **Exclude Audience Network from Sales campaigns.** Or create a separate AN adset on a reach KPI and budget it deliberately.
- **Rate limit hygiene:** one diagnostic pass per session; batch all reads into a single call; never loop `/act_X/ads` listings.

## Doc log

- **2026-06-25** - Initial consolidation.
- **2026-06-25** - Merged targeting-replacement pitfalls, Advantage+ locks, field-stripping recovery, safe-edit checklist, CAPI dedup patterns.
