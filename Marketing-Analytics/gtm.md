# Google Tag Manager (GTM)

**Use for:** Tag deployment and event tracking on websites; intermediary between the site and GA4 for custom purchase event configuration.
**Status:** Active

## Setup & access

- No official Google GTM MCP exists. Stape is the most credible third-party option evaluated.
- Direct GTM container access (read/edit tags, triggers, variables) is available via the Tag Manager API v2 + direct REST (no MCP needed). See scars for the workspace lifecycle gotcha and required OAuth scopes.
- A common WordPress/WooCommerce setup runs GTM via the **GTM4WP** plugin.

## Scars & gotchas

- **A gallery template's tag `type` is `cvt_<galleryTemplateId>`, NOT the local `templateId`** - Cloning a gallery tag (e.g. Microsoft Clarity) into another container: `POST .../templates` with the source template's `galleryReference` + `templateData` succeeds and returns a local `templateId` (e.g. `12`), but creating the tag with `type: "cvt_<containerId>_12"` fails with `vendorTemplate.key: Unknown entity type`. The correct type is the gallery id from `galleryReference.galleryTemplateId`, i.e. the SAME string in every container (e.g. `cvt_MQDKZ` for Clarity). Copy the `type` verbatim from the source container's tag.

- **`consentSettings.consentType` is a single Parameter of type `list`, not a repeated field** - Setting a tag's consent requirement via the API, the obvious `consentType: [{...}]` returns `Unknown name "consentType": Proto field is not repeating, cannot start list`, which reads like the field doesn't exist. The working shape is `consentSettings: { consentStatus: "needed", consentType: { type: "list", list: [{ type: "template", value: "analytics_storage" }] } }`. Omitting `consentType` entirely gives `consentSettings.type: Parameter must have a type`.

- **Creating a container needs only `tagmanager.edit.containers`; versioning needs `edit.containerversions`** - `POST .../accounts/<id>/containers` succeeds without the `tagmanager.manage.accounts` scope (contrary to expectation), so a service account with edit+publish can stand up a whole new container. But `workspaces:create_version` returns 403 `Insufficient Permission` unless `tagmanager.edit.containerversions` is in the requested scopes. Request all three (`edit.containers`, `edit.containerversions`, `publish`) up front.

- **Microsoft Clarity ignores Google Consent Mode - gate it in GTM or it records deniers** - Clarity is not a Google tag, so a Consent Mode default of `analytics_storage: denied` does nothing to it: the tag still loads and records the session. This can run unnoticed for weeks, so EU/UK visitors who clicked deny on the banner are still being session-recorded. Fix is a GTM consent requirement on `analytics_storage` (see the shape scar above); Clarity's own "cookie consent" setting is a second layer, not a substitute, because it doesn't stop the script loading.

- **Publishing a stale workspace silently reverts newer changes, and the committed snapshot will not tell you** - GTM raises no conflict when you version-and-publish a workspace that predates the current live version: it just overwrites. On one site an unnamed version was published from a workspace created before a later live version, which reset a consent-gated tag from `consentStatus: needed` back to `notSet`. The live site then session-recorded EU/UK visitors who had explicitly denied `analytics_storage`, for about a day, with **nothing in the repo showing it**: the newest committed snapshot correctly described the intended state, while the drift lived only in GTM. A snapshot proves what you *published*, never what is *live*. Two habits, both cheap: **name every version** (an unnamed version is indistinguishable from a stray republish), and **fetch `containers/<id>/versions:live` and diff it against the newest committed snapshot** whenever you touch a container or audit one. Commit the drifted version too, so the history is honest.

- **Default Workspace looks empty (stale) after publishing from a separate workspace via the API** - The GTM API rejects edits to a workspace that's been submitted/published ("Workspace is already submitted"), so each scripted change set needs a FRESH workspace (`POST .../workspaces`), and publishing consumes it. GTM then auto-deletes those throwaway workspaces, leaving the human-facing **Default Workspace behind the live version** - the Triggers/Tags lists render empty even though the LIVE container has everything. Don't panic: check `versions:live` to confirm the published container, then `POST .../workspaces/<id>:sync` to rebase the Default Workspace onto the live version so the UI matches. Confirmed on a live site: the live version had the trigger+tags; the Default Workspace showed none until synced.

- **GA4 purchase-event mismatch: revenue can land inconsistently** - A common open failure mode: GTM-fired GA4 purchase events produce revenue that lands between store net and store net+shipping, rather than cleanly at one or the other. Likely causes: GTM tag config, dataLayer push shape, or GA4 ecommerce schema. Do not treat GA4 revenue as exactly net or exactly gross without checking the current mapping first.

- **No official Google GTM MCP: Stape is the best available third-party option** - Stape's credibility in the GTM ecosystem makes it the default if programmatic GTM access via an MCP is needed. Validate against current Stape docs before wiring (things change).

- **GTM changes must be version-controlled and scripted, not applied manually in the UI** - Execute every container mutation via a scripted apply tool (e.g., `gtm-apply.mjs`) rather than the GTM UI, and commit container snapshots to a backups folder after each change. This gives an auditable, reproducible history of all tag and trigger changes. Required OAuth scopes for API writes: `tagmanager.edit.containers`, `tagmanager.edit.containerversions`, `tagmanager.publish`.

- **"Lead = form submit OR WA click" is a GTM-level decision that double-counts Meta Leads** - A container that fires `fbq('track', 'Lead')` on both form-submit and WhatsApp-button-click triggers maximizes reported Leads for Meta optimization but inflates Meta's Lead count well above true CRM lead count. Cleaner pattern: split WA clicks to fire `fbq('track', 'Contact')` instead of `Lead` so the two metrics can be compared cleanly.

- **Domain-wide delegation can drive the Tag Manager API when Workspace blocks third-party OAuth** - A service account with domain-wide delegation can call the GTM API (CRUD tags/triggers and publish container versions) without requiring a user to authorize third-party OAuth scopes in Google Workspace. Required scopes: `tagmanager.edit.containers`, `tagmanager.edit.containerversions`, `tagmanager.publish`.

- **A GA4 Event tag created via the Tag Manager API needs `measurementIdOverride`, not `measurementId`** - when creating a GA4 Event tag (type `gaawe`) through the API, the destination-id parameter key is `measurementIdOverride`. Passing `measurementId` is ignored, and the tag ships with no destination, so it fires and sends nowhere. Set the tag parameter name to `measurementIdOverride` with the `G-XXXXXXX` value.

## Conclusions / best practices

- **Always document the GTM-to-GA4 purchase event mapping per property** before building cross-source reports. Specifically: which dataLayer event fires, what revenue field it pushes, and how GA4's ecommerce schema receives it.
- **Resolve any GA4/GTM purchase-event mismatch** before the next report that relies on GA4 revenue as the attribution numerator. The path is: verify GTM tag config -> confirm dataLayer `purchase` event fields -> confirm GA4 ecommerce schema -> pick net vs net+shipping as the canonical mapping.
- **For direct Tag Manager API access, drive the Tag Manager API v2 REST directly** (no MCP needed) and version-control the container JSON. Use Stape only if an MCP layer is specifically wanted; re-check its current API coverage before wiring.
- **Script and version-control all GTM changes.** Use a scripted apply tool (e.g., `gtm-apply.mjs`) and commit container snapshots after each mutation. Manual UI edits leave no auditable trail.
- **Separate event names by intent.** Firing `Lead` on both form submits and WA clicks maximizes Meta optimization signal but breaks clean CRM reconciliation. Where lead quality matters, split WA clicks to `Contact` and reserve `Lead` for actual form submits.
