# Google Tag Manager (GTM)

**Use for:** Tag deployment and event tracking on websites; intermediary between the site and GA4 for custom purchase event configuration.
**Status:** Active

## Setup & access

- No official Google GTM MCP exists. Stape is the most credible third-party option evaluated.
- Direct GTM container access (read/edit tags, triggers, variables) is available via the Tag Manager API v2 + direct REST (no MCP needed). See scars for the workspace lifecycle gotcha and required OAuth scopes.
- A common WordPress/WooCommerce setup runs GTM via the **GTM4WP** plugin.

## Scars & gotchas

- **Default Workspace looks empty (stale) after publishing from a separate workspace via the API** - The GTM API rejects edits to a workspace that's been submitted/published ("Workspace is already submitted"), so each scripted change set needs a FRESH workspace (`POST .../workspaces`), and publishing consumes it. GTM then auto-deletes those throwaway workspaces, leaving the human-facing **Default Workspace behind the live version** - the Triggers/Tags lists render empty even though the LIVE container has everything. Don't panic: check `versions:live` to confirm the published container, then `POST .../workspaces/<id>:sync` to rebase the Default Workspace onto the live version so the UI matches. Confirmed on knightica.com (2026-07-24): live version had the trigger+tags; the Default Workspace showed none until synced.

- **GA4 purchase-event mismatch: revenue can land inconsistently** - A common open failure mode: GTM-fired GA4 purchase events produce revenue that lands between store net and store net+shipping, rather than cleanly at one or the other. Likely causes: GTM tag config, dataLayer push shape, or GA4 ecommerce schema. Do not treat GA4 revenue as exactly net or exactly gross without checking the current mapping first.

- **No official Google GTM MCP: Stape is the best available third-party option** - Stape's credibility in the GTM ecosystem makes it the default if programmatic GTM access via an MCP is needed. Validate against current Stape docs before wiring (things change).

- **GTM changes must be version-controlled and scripted, not applied manually in the UI** - Execute every container mutation via a scripted apply tool (e.g., `gtm-apply.mjs`) rather than the GTM UI, and commit container snapshots to a backups folder after each change. This gives an auditable, reproducible history of all tag and trigger changes. Required OAuth scopes for API writes: `tagmanager.edit.containers`, `tagmanager.edit.containerversions`, `tagmanager.publish`.

- **"Lead = form submit OR WA click" is a GTM-level decision that double-counts Meta Leads** - A container that fires `fbq('track', 'Lead')` on both form-submit and WhatsApp-button-click triggers maximizes reported Leads for Meta optimization but inflates Meta's Lead count well above true CRM lead count. Cleaner pattern: split WA clicks to fire `fbq('track', 'Contact')` instead of `Lead` so the two metrics can be compared cleanly.

- **A GA4 Event tag created via the Tag Manager API needs `measurementIdOverride`, not `measurementId`** - when creating a GA4 Event tag (type `gaawe`) through the API, the destination-id parameter key is `measurementIdOverride`. Passing `measurementId` is ignored, and the tag ships with no destination, so it fires and sends nowhere. Set the tag parameter name to `measurementIdOverride` with the `G-XXXXXXX` value.

## Conclusions / best practices

- **Always document the GTM-to-GA4 purchase event mapping per property** before building cross-source reports. Specifically: which dataLayer event fires, what revenue field it pushes, and how GA4's ecommerce schema receives it.
- **Resolve any GA4/GTM purchase-event mismatch** before the next report that relies on GA4 revenue as the attribution numerator. The path is: verify GTM tag config -> confirm dataLayer `purchase` event fields -> confirm GA4 ecommerce schema -> pick net vs net+shipping as the canonical mapping.
- **For direct Tag Manager API access, drive the Tag Manager API v2 REST directly** (no MCP needed) and version-control the container JSON. Use Stape only if an MCP layer is specifically wanted; re-check its current API coverage before wiring.
- **Script and version-control all GTM changes.** Use a scripted apply tool (e.g., `gtm-apply.mjs`) and commit container snapshots after each mutation. Manual UI edits leave no auditable trail.
- **Separate event names by intent.** Firing `Lead` on both form submits and WA clicks maximizes Meta optimization signal but breaks clean CRM reconciliation. Where lead quality matters, split WA clicks to `Contact` and reserve `Lead` for actual form submits.

## Doc log

- **2026-06-25** - Initial consolidation.
- **2026-06-25** - Added workspace-sync scar, gtm-apply.mjs scripting pattern, Lead/Contact event split issue.
