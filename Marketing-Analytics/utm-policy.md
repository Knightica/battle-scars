# UTM Tagging Policy (Knightica standard)

**Use for:** Every campaign/placement link Knightica hands out - its own marketing and client engagements alike. One consistent scheme so GA4 attribution never fragments and any team member (or an agent generating a link on request) produces the same result.

**Status:** Active
**Applies to:** knightica.com + all client GA4 properties
**Last validated:** 2026-07-24

---

## 0. How to generate a link (the deterministic method)

Given a use case, produce the URL in five steps:

1. **Destination URL** - the exact page (usually the homepage; a specific landing page if the campaign points there). Keep any existing path; append params with `?` (or `&` if the URL already has a `?`).
2. **Pick the scenario** from §3 -> this fixes `utm_source` + `utm_medium` (never freestyle these).
3. **`utm_campaign`** - the specific initiative, as a lowercase hyphen-slug (§4).
4. **`utm_content` / `utm_term`** - only if they add signal (variant / person / step; paid-search keyword).
5. **Assemble** in order: `utm_source, utm_medium, utm_campaign, utm_content, utm_term`. URL-encode values. Done.

Example use case -> link:
> "LinkedIn ad for the summer webinar, landing on the contact page."
> `https://knightica.com/contact/?utm_source=linkedin&utm_medium=paid_social&utm_campaign=summer-webinar`

---

## 1. The five parameters

| Param | Required | Meaning |
|---|---|---|
| `utm_source` | yes | Where the click originates (platform / publication / partner / list). |
| `utm_medium` | yes | The channel *type*. Drives the GA4 default-channel grouping. |
| `utm_campaign` | recommended | The specific initiative. |
| `utm_content` | optional | Distinguish variants: which ad, which link, which person, which sequence step. |
| `utm_term` | optional | Paid-search keyword (usually auto-tagged; rarely manual). |

---

## 2. Rules

- **Lowercase everything. No spaces.**
- **`source` and `medium` are a controlled vocabulary** (§3) - never invent new values. A stray `utm_medium=Social` or `utm_source=FB` splits the channel and corrupts reporting.
- **`campaign` / `content` / `term` are hyphen-slugs**: lowercase, spaces -> hyphens, strip punctuation. e.g. `summer-launch`, `q3-intro`, `2026-08-newsletter`.
- **`source`/`medium` tokens use underscores** where multi-word (`paid_social`, `email_signature`) - that's the platform-recognized form; don't hyphenate them.
- Don't tag **internal links** (same-site navigation) - it resets the session/attribution. UTMs are for *inbound* links only.
- Don't manually UTM **Google Ads / YouTube Ads** when auto-tagging (gclid) is on - it double-tags. Use this policy for everything auto-tagging doesn't cover.

---

## 3. Controlled vocabulary - `utm_medium` -> GA4 channel

Get `medium` right and the link lands in the correct GA4 default channel automatically.

| `utm_medium` | GA4 default channel | Use for |
|---|---|---|
| `social` | Organic Social | owned profile/bio links, organic posts |
| `paid_social` | Paid Social | Meta / Instagram / LinkedIn **ads** |
| `cpc` | Paid Search | Google Search ads (manual) |
| `video` | Organic/Paid Video | YouTube (organic or ad) |
| `email` | Email | newsletter sends **and** email signatures |
| `referral` | Referral | partners, directories, other sites |
| `pr` | custom group* | earned media / press |
| `outreach` | custom group* | cold-outreach funnels |
| `qr` | custom group* | QR codes on physical media |
| `print` | custom group* | printed ads |

\* `pr`, `outreach`, `qr`, `print` are not in GA4's default channel group - they land in **"Unassigned"** unless you add a **custom channel group** (Admin -> Data display -> Channel groups). Recommended one-time group: rule `Session manual medium matches regex ^(pr|outreach|qr|print|offline)$` -> name it e.g. "Earned / Direct-response". Custom groups apply retroactively within the retention window.

`utm_source` values (the origin): `instagram` · `facebook` · `linkedin` · `youtube` · `tiktok` · `x` · `github` · `telegram` · `whatsapp` · `google` · `newsletter` · `email_signature` · `linktree` · `<publication-slug>` · `<partner-slug>` · `<list-slug>` (outreach list).

---

## 4. Per-scenario recipes

| Scenario | `utm_source` | `utm_medium` | `utm_campaign` | `utm_content` |
|---|---|---|---|---|
| Owned social bio / profile link | `<platform>` | `social` | `profile` | - |
| Organic social post | `<platform>` | `social` | `<post-slug>` | - |
| Meta ad | `facebook` / `instagram` | `paid_social` | `<campaign>` | `<ad-variant>` |
| LinkedIn ad | `linkedin` | `paid_social` | `<campaign>` | `<ad-variant>` |
| YouTube ad | `youtube` | `video` | `<campaign>` | `<ad-variant>` |
| Google Search ad (manual) | `google` | `cpc` | `<campaign>` | - |
| Newsletter send | `newsletter` | `email` | `<send-slug>` (e.g. `2026-08-launch`) | - |
| Email signature | `email_signature` | `email` | `signature` | `<person>` |
| PR / press | `<publication>` | `pr` | `<publication>_<yyyy-mm>` | - |
| Cold outreach | `<list-slug>` | `outreach` | `<funnel>` | `<sequence-step>` |
| Partner / directory | `<partner>` | `referral` | optional | - |
| Print ad | `<placement>` | `print` | `<campaign>` | - |
| QR code | `<location-or-material>` | `qr` | `<campaign>` | - |
| Linktree | `linktree` | `social` | `bio` | - |

---

## 5. Examples

- Instagram bio -> `?utm_source=instagram&utm_medium=social&utm_campaign=profile`
- Meta retargeting ad, "efficiency" creative -> `?utm_source=facebook&utm_medium=paid_social&utm_campaign=q3-retargeting&utm_content=efficiency`
- Newsletter, August launch -> `?utm_source=newsletter&utm_medium=email&utm_campaign=2026-08-launch`
- A team member's email signature -> `?utm_source=email_signature&utm_medium=email&utm_campaign=signature&utm_content=<person>`
- Cold outreach email, partner list, step 2 -> `?utm_source=partners-list&utm_medium=outreach&utm_campaign=q3-intro&utm_content=step-2`
- QR on a conference flyer -> `?utm_source=techconf-tlv&utm_medium=qr&utm_campaign=techconf-2026`

---

## Scars & gotchas

- **Custom mediums (`pr`, `outreach`, `qr`, `print`) show as "Unassigned"** until a custom channel group is created; the data is captured correctly, only the default Channels report mislabels it. _(carried from ga4.md)_
- **One inconsistent `source`/`medium` fragments a channel** - the whole point of the fixed vocabulary. Ad platforms auto-generating campaign-name variants is a known upstream fragmenter (see `meta-ads.md`). _(carried from ga4.md)_
