# battle-scars

**Hard-won, platform-specific lessons from shipping real automations in production.**

Every file here is a scar. Something broke in a live system, we figured out why, and we wrote down the fix that held. Not the happy path. The landmines. The stuff the official docs don't warn you about until it has already cost you a weekend.

```
// the docs tell you how it's supposed to work.
// this repo tells you how it actually breaks.
```

## Who we are

We're **[Knightica](https://knightica.com)**, an AI automation agency. We help businesses run leaner and stay ahead of change with modern technology. In practice that means we wire a lot of third-party platforms together, payments, messaging, CRM, analytics, hosting, and every one of them has its own way of biting you.

This repo is what we learned, cleaned up and made public.

## Why we're sharing it

We believe in **simplicity** and **honest communication**, so keeping this locked in a private wiki felt wrong. Integration knowledge usually dies in Slack threads and postmortems. These lessons are reusable and carry nothing client-specific, so there's no reason not to publish them.

If one bullet in here saves you a production incident, it did its job. That's the whole point.

## How to read a file

Each file assumes you already know roughly how the platform works and just need the sharp edges. Pair it with the vendor's own docs the first time. Two things worth knowing:

- **These capture what breaks, not how to start.** A landmine map, not a tutorial.
- **Live platforms drift.** APIs, dashboards, and pricing change. Treat every scar as a snapshot of a live platform: re-verify against current docs before betting on it, and open an issue if it no longer holds.

Start with **[`PATTERNS.md`](PATTERNS.md)**, the platform-agnostic rules (idempotency, fail-closed payments, phone normalization, queue isolation, secret hygiene). They hold no matter what you're wiring. Then jump to the platform you're touching.

## Index

### Payments
| Platform | What bites |
|---|---|
| [PayPlus](Payments/payplus.md) | Unsigned IPN, dashboard-vs-per-request double-fire, iframe checkout, phone-home to verify, refund endpoint, IP-allowlist 1010, no-charge standing orders fire no IPN |
| [Rivhit](Payments/rivhit.md) | Duplicate-invoice incident, misleading doc-type env var, credit notes for refunds |
| [Greeninvoice](Payments/greeninvoice.md) | VAT-on-top math, doc type 320, undocumented webhook HMAC, two events per payment, no webhook-management API, sequential-date rule, preview as validator |

### Messaging
| Platform | What bites |
|---|---|
| [ManyChat](Messaging/manychat.md) | 1 req/sec limit + blast starvation, dup-subscriber recovery, `phone_lookup` you must stamp yourself, template review resets delivery, `sendFlow` 200 is not delivery, WhatsApp cannot carry a hidden ref |
| [Telegram](Messaging/telegram.md) | Supergroup-only reactions, reaction rate limit, 24h getUpdates buffer, capturing a chat id under Privacy Mode, a bot never sees its own sends in getUpdates |
| [Green API (WhatsApp)](Messaging/green-api-whatsapp.md) | Phone chat-id format, silent group add/remove failures, Meta's account-level spam guard hits any unofficial bridge, blast pacing + resend ledger |

### Marketing & analytics
| Platform | What bites |
|---|---|
| [Meta Ads](Marketing-Analytics/meta-ads.md) | Targeting full-replacement, Advantage+ edit locks, field-strip deletes audiences, immutable ad-set fields, `end_time` must be epoch, `spend_cap` unit trap + silent shutdown, MER vs ROAS |
| [GA4](Marketing-Analytics/ga4.md) | SA access via Admin API, custom channel group replaces (not extends) the default, key events pre-fire, Realtime double-counts MP, MP drops malformed session_id behind a 2xx |
| [GTM](Marketing-Analytics/gtm.md) | No official MCP, version-controlled container via API, stale workspace publish reverts newer changes, gallery-template `type` quirk, consent gating gaps, Lead double-count inflation |
| [Google Ads](Marketing-Analytics/google-ads.md) | Developer token lives on the manager account, conversion category/biddable traps, GA4 imports arrive HIDDEN, API version 404-vs-deprecation, Maximize Conversions cold-start CPC blowup, ad-review crawler fakes paid traffic in GA4 |
| [UTM policy](Marketing-Analytics/utm-policy.md) | A full tagging standard: source/medium vocabulary to GA4 channels, per-scenario recipes, deterministic build method |

### Automation
| Platform | What bites |
|---|---|
| [Trigger.dev](Automation/trigger-dev.md) | One project per codebase, syncEnvVars or silent failures (and sourced-.env quote stripping), free-plan cap freeze, node-22, dev key parks runs QUEUED forever, scoped keys break realtime tokens and read-only checks, bulk actions + health report |
| [n8n](Automation/n8n.md) | Crypto node has no `require`, webhook-path secrets, double-`+` phone bug, a disconnected trigger returns 200 and drops every payload |

### CRM
| Platform | What bites |
|---|---|
| [Monday](CRM/monday.md) | Per-field rate-limit surge, label-id not index, board_relation query gap, contact-vs-lead dedup, checkbox read/write asymmetry, dashboards bind at creation only, outage signature |
| [Calendly](CRM/calendly.md) | iCalUID + organizer dedup layers, cancellation handling, polling not webhooks |
| [Cal.com](CRM/cal-com.md) | Webhook-driven, wrapped payload, x-cal-signature-256 HMAC over raw body, embed metadata never forwarded (use hidden booking questions), notes not at `payload.notes`, preview-URL webhook goes dark silently |
| [Arbox](CRM/arbox.md) | Read-only user API (405), phone corruption, pagination via next_page_url, membership report is a live snapshot, UI ids vs API ids are different spaces |

### Infra
| Platform | What bites |
|---|---|
| [Supabase](Infra/supabase.md) | RLS recursion + deny-by-default, enable RLS explicitly on server-only tables, MCP unauth for DDL, node-22 Realtime, private buckets, PostgREST silently caps reads at 1000 rows |
| [Netlify](Infra/netlify.md) | MCP silent failures, build OOM, raw-body HMAC, redirect globs are no-ops, ignore-rule quirk, secret env vars are deploy-bound, free-tier badge injection, site-count ceiling, geo-consent edge function, `config.rateLimit` not enforced |
| [Cloudways](Infra/cloudways.md) | MX records are dashboard-only, ANAME quirks, CNAME needs explicit `ttl`, SSH ForceCommand (use Git Deploy), TTL caching on SSL re-verify |
| [Webflow](Infra/webflow.md) | Client-owned site goes dark with zero errors in your stack, unverified domain 404s every path, form webhook orphaned by host migrations |
| [Lovable](Infra/lovable.md) | Exports ship dead hostnames, invented date logic, weekday hardcoded in copy, a second unmanaged Supabase PII store with a public key in the bundle |
| [GitHub](Infra/github.md) | Org-vs-personal placement, async transfers + remote re-pointing, deploy keys, CI env-var skip, CODEOWNERS + access-model traps, force-push does not purge orphaned commits, Git Data API server-side restore |

### Google
| Platform | What bites |
|---|---|
| [Google Workspace APIs](Google/workspace-apis.md) | DWD scope-block behavior, SA-key org policy, calendar.events for deletes, Hebrew subject encoding, primary-address rename breaks sending, deleting a user deletes every Drive file they own |

### Ecommerce
| Platform | What bites |
|---|---|
| [WooCommerce](Ecommerce/woocommerce.md) | IL VAT/currency config, Polylang over TranslatePress, GA4 revenue mapping, WAF wc/v3 namespace bypass, read-only keys 401 on writes, `urllib` WAF 403, control chars in order JSON, real sales-per-coupon |

### Legal
| Platform | What bites |
|---|---|
| [Net HaMishpat + Israeli Legal Aid](Legal/net-hamishpat.md) | No public court API, smart-card-bound access, five-vendor court-sync market with a Legal Aid gap, statutory fee schedule, Israeli cloud/confidentiality position |

### AI
| Platform | What bites |
|---|---|
| [Claude Code (agent tooling)](AI/claude-code.md) | Claude in Chrome silently dead in Arc (connects, every command times out), dedicated Chrome agent-browser pattern, localhost preview cache, screenshot-after-navigate timing, WebKit fallback for mobile checks |
| [Claude API (Anthropic)](AI/claude-api.md) | JSON code-fence stripping, Hebrew transliteration fallback, maxRetries bump |
| [Hermes Agent (Nous Research)](AI/hermes-agent.md) | Always-on agent daemon: MCP tools ungated, no RBAC, container backends skip the approval gate, one poller per bot, marketplace image skips your SSH key, unofficial WhatsApp bridge, memory files hard-capped and never auto-compact |

### Cross-cutting
| Doc | Why |
|---|---|
| [PATTERNS.md](PATTERNS.md) | Platform-agnostic scars: reliability, deploy/config, data hygiene, payments, robustness, secret hygiene, email deliverability, monitoring, rendering/print/RTL, consent gating |
| [GLOSSARY.md](GLOSSARY.md) | Shorthand whose meaning here isn't the textbook one (MER, soft-skip, fail-closed, mirror column) |

## A note on scope

These are field notes, not a certification. The platforms here **skew toward the Israeli business ecosystem** (local payment providers, invoicing, CRM, WhatsApp-first messaging, the courts), **but plenty are international** too, GA4, GTM, Google Ads, Meta, Supabase, Netlify, Trigger.dev, Cal.com, Webflow, Lovable, Claude, Hermes Agent. Some lessons are therefore regional (VAT, Hebrew handling, local vendors) and some are specific to the stack we run. Take what applies, leave what doesn't.

Found a mistake, or have a scar of your own? Open an issue or a PR. We'd rather be corrected than confidently wrong.

## License

Published under [CC BY 4.0](LICENSE). Use it, quote it, build on it.

```
// built from real projects, sanitized of anything client-specific.
```

Made with scars by the Knightica team.

**Make the Right Move.**
