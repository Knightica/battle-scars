# Webflow

**Use for:** Client marketing sites and landing pages built visually by the client, with native form submissions posted to a backend automation bridge.
**Status:** Active (client-owned)

**TL;DR:** Webflow sites you touch are usually owned and edited by the client, so the failure modes are organizational as much as technical: a site gets unpublished, a plan lapses, a domain gets detached, and the funnel that depends on its forms goes dark without a single error anywhere in your own stack. Verify the site is actually serving before debugging anything downstream.

## Setup & access

- Custom domains need three DNS records, and Webflow checks all of them: an `A` at the apex pointing to Webflow's IP, a `CNAME` on `www` to `cdn.webflow.com`, and a `TXT` at `_webflow` carrying the one-time verification value from the site's domain settings.
- The `_webflow` TXT is the one people forget. Without it the domain sits at "Unverified domain / Not published" and Webflow serves its own 404 page for the domain, no matter how correct the A and CNAME are.
- Form submissions go out as a native webhook: `POST` with `x-webflow-signature` (HMAC-SHA256) and `x-webflow-timestamp`. Verify the signature with a proper crypto step in whatever automation tool receives it, not a hand-rolled comparison in a generic code step (HMAC verification needs a constant-time compare, easy to get subtly wrong by hand).
- The Webflow MCP cannot edit inside components, so field additions to a shared form component are a manual Designer job for the client.

## Scars & gotchas

- **A Webflow domain that 404s on every path is unverified or unpublished, not a DNS problem** - A client domain returned 404 on `/`, `/sitemap.xml`, `/robots.txt`, and every guessed page. DNS was perfect (A -> Webflow's IP, `www` -> `cdn.webflow.com`), which sent the first pass of debugging in the wrong direction. The tell is in the response headers: `x-wf-region` and a `surrogate-key` containing `404req` mean Webflow is answering and refusing to serve the site, so the problem is inside Webflow, not the DNS. The dashboard showed "Unverified domain - Not published" and was waiting on the `_webflow` TXT record. Check the site's domain settings before touching a single DNS record.

- **A client-owned Webflow site can silently take a funnel down for months** - A webinar funnel had zero signups for three months. Two causes stacked: the Webflow site had stopped serving (above), and the form's webhook still pointed at a hostname that had gone NXDOMAIN during a domain migration. Neither produced an error anywhere downstream, because nothing ever arrived. When a funnel's numbers are zero rather than wrong, check that the front door is open before auditing the pipeline.

- **Domain migrations orphan the form webhook** - Webflow's form webhook URL lives in the Webflow dashboard, not in your repos, so a host change leaves it pointing at the old hostname and every submission fails silently. Webflow is one of several dashboard-only places to update on any host migration, alongside the payment provider's webhook and any chat-platform "external request" URLs. Keep them on one checklist.

## Conclusions / best practices

- **Verify the site serves before debugging downstream.** One `curl -sI https://<domain>/` costs nothing. `x-wf-region` in the headers plus a 404 body means Webflow is refusing to serve, not that the route is wrong.
- **Owning the DNS doesn't mean DNS is the fault.** If you manage the zone yourself, DNS is the tempting explanation because it's the thing you can fix in seconds. Check the platform's own domain status first.
- **Treat client-owned front ends as an availability risk.** If a funnel's entry point lives somewhere the client can unpublish, add an uptime check on it, or move the landing page to a repo and hosting you control, independent of the Webflow site.
- **One checklist for host migrations:** the marketing-site form webhook, the payment-provider webhook, and any chat-platform external-request URLs. All of these are typically dashboard-only and none of them live in git.
