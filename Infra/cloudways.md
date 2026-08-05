# Cloudways

**Use for:** DNS management via the DNS Made Easy MCP (Cloudways-hosted zones). Primary tool for adding CNAMEs, ANAMEs, TXT records, and MX records on domains.
**Status:** Active

## Setup & access

- DNS Made Easy is the DNS provider under the Cloudways account. Manage via the `dns_made_easy` MCP tools where possible.
- For records where the MCP falls short (see scars), fall back to the DNS Made Easy dashboard directly.
- Propagation can be verified against the authoritative NS (digicertdns.*) and via 1.1.1.1 / 8.8.8.8.

## Scars & gotchas

- **`dns_made_easy_add_records` rejects empty `name` for apex records** - The MCP advertises `name: ""` for the root domain, but the Cloudways wrapper returns `422 ... name must be a string` on an empty string. Pass `"@"` instead - the API normalizes it back to `name: ""` (true apex) on storage. Confirmed adding an apex SPF TXT record.

- **2048-bit DKIM TXT is auto-chunked into 255-char quoted strings** - Adding a Google Workspace 2048-bit DKIM key (`v=DKIM1; k=rsa; p=...`, ~400 chars) via `dns_made_easy_add_records` succeeds as a single `value`; DNS Made Easy splits it into two quoted strings on serve (`"...part1" "...part2"`). Resolvers concatenate them, Google reads it fine - no manual splitting needed.

- **DigiCert DNS = DNS Made Easy** - Domains on `ns*.digicertdns.com/.net` nameservers (DigiCert acquired DNS Made Easy) are managed through the same Cloudways `dns_made_easy` MCP. whois may show a different registrar plus digicertdns nameservers, but the zone is editable via Cloudways with no separate DigiCert login.

- **DNS Made Easy MCP drops MX priority (mx_level field)** - When adding MX records via the MCP, `mx_level` is silently omitted. The record is created without the correct priority value. Add MX records manually in the DNS Made Easy dashboard whenever priority matters.

- **MX via the MCP is now fully dead, not just lossy - every bypass fails** - As of MCP server v1.28.1 the Cloudways API hard-rejects MX without priority (`422: The records.0.mx level field is required`), so the record is not created at all. Every workaround is exhausted: raw JSON-RPC `tools/call` to the MCP endpoint with `mx_level`/`mxLevel`/`priority`/`level`/`mx_priority` all stripped server-side (FastMCP schema validation; `tools/list` confirms no priority field exists in the schema); priority prefixed in the value (`"1 smtp.google.com."`) rejected as an invalid domain; and the public Cloudways API (`api.cloudways.com/api/v1`) has no DNS Made Easy endpoints at all. The DNS Made Easy dashboard is the ONLY way to create an MX record. Everything else (A, ANAME, CNAME, TXT, SPF) works fine via the MCP.

- **Use apex ANAME over A records** - DNS Made Easy supports ANAME (a flattened CNAME for apex domains). Prefer it over static A records for apex entries pointing at Netlify. ANAME follows the load balancer IP dynamically; A records break if Netlify's IPs rotate.

- **Missing SPF/DMARC causes confirmation emails to land in spam** - Having Google MX and DKIM is not enough. Without SPF and DMARC, outbound transactional emails from the domain reliably land in spam. Add `v=spf1 include:_spf.google.com ~all` (TXT on the apex) and a DMARC record (`v=DMARC1; p=none` minimum) at the same time you wire Google Workspace DNS.

- **SSH interactive shell is disabled for Cloudways app users via ForceCommand** - Even with a valid deploy key, Cloudways blocks interactive SSH sessions for app users. rsync-over-SSH will fail. Switch to the Cloudways Git Deployment feature instead: wire a GitHub deploy key (read-only), enable `deploy_keys_enabled_for_repositories` on the org if needed, and trigger deploys via the Cloudways REST API. The switch takes about 5 minutes and requires no support ticket.

- **Self-host short hero MP4 loops on the Cloudways CDN, not YouTube/Vimeo** - Embedding a third-party video player adds ~200KB+ of iframe overhead, imposes branded player chrome, and causes autoplay flakiness on mobile due to browser/platform restrictions. Cloudways' built-in CDN handles short looping MP4 files fine: full control, no tracking, no ads, deterministic autoplay behavior.

- **Isolate live ecommerce on its own Cloudways server** - Do not bundle an active ecommerce site with staging, preview, or brochure sites on the same server. Noisy-neighbor effects (traffic spikes, memory pressure from adjacent sites) can directly impact checkout reliability. Staging and preview can share a cheaper server; production commerce needs isolation.

- **DNS Made Easy ANAME rejected via API despite enum support - but the rejection is not universal** - When creating an ANAME record for an apex domain via the DNS Made Easy API, the record type may be rejected even though `ANAME` appears in the supported type enum. On a later apex (with name `"@"`) the same ANAME create succeeded first try via the MCP, so try ANAME first; if it 422s, fall back to an A record with Netlify's published IP (75.2.60.5) for the apex and a CNAME for www, or create the ANAME via the dashboard.

- **TTL caching delays SSL re-verification after a DNS record change** - After correcting a DNS record (e.g., fixing a CNAME value needed for SSL verification), Netlify's resolver may cache the prior incorrect value for ~5 minutes despite the record being correct in the zone. Lower TTL to 60 seconds before making any verification-critical record changes, and wait for propagation to both 1.1.1.1 and the authoritative NS before re-triggering SSL issuance.

- **Apex DNS cutover to Netlify: A + www CNAME, preserve all existing records** - When cutting an apex domain over to Netlify (A record on Netlify's published IP 75.2.60.5 plus a www CNAME to the `*.netlify.app` target), preserve every existing record in place: any subdomain CNAMEs, Google MX, DKIM, SPF, and platform verification TXT records. Template for any cutover to Netlify where a Google Workspace org is already live on the domain.

## Conclusions / best practices

- Prefer ANAME for apex entries pointing at Netlify (or any CDN that publishes a hostname rather than stable IPs). If the DNS Made Easy API rejects the ANAME, create it via the dashboard, or fall back to an A record on Netlify's published IP (75.2.60.5) plus a www CNAME.
- MX records are dashboard-only: the MCP/API path hard-fails on the missing priority field and no bypass exists. Create MX in the DNS Made Easy dashboard and verify `mx_level` there.
- When wiring Google Workspace DNS, ship SPF + DMARC alongside MX and DKIM in the same session. Deferring them means transactional email is silently unreliable until a deliverability audit catches it.
- After adding or updating records, verify propagation against both the authoritative NS and a public resolver (1.1.1.1 or 8.8.8.8) before signing off.
