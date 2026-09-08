# Net HaMishpat (נט המשפט) and the Israeli Legal Aid Department

**Use for:** Any automation touching Israeli court filings, case data, or Legal Aid (הסיוע המשפטי) fee claims. Read this before promising a client any court-system integration.
**Status:** Active (research-stage, no build yet)

## Setup & access

- **Net HaMishpat** (NET) is the Israeli courts' institutional system: ongoing proceedings, filings, decisions, hearing dates. Representing lawyers authenticate with a smart card (כרטיס חכם), a hardware-bound personal credential.
- **The Legal Aid Department** (LAD, הסיוע המשפטי) runs its own closed government web interface, used to report actions and claim the associated fee. Its fee-reporting process is not documented publicly.
- Neither system publishes an API for ordinary use.

## Scars & gotchas

- **There is NO public API for Net HaMishpat, and there is no way around the smart card.** A gated data interface exists but has been granted only to commercial legal databases (Nevo, Takdin) under commitment letters with the Courts Administration; civil-society requests for equivalent access have not opened it. Do not scope court-system integration as an API job.

- **The Israeli practice-management vendors are NOT riding a privileged interface, they drive the lawyer's own session from a workstation with the card inserted.** Masig documents it outright: the NET connection "will be activated if you possess a smart card" and the module "can be activated from any workstation provided a smart card is connected to that workstation." If a vendor serving roughly 20,000 lawyers still needs a card in a reader on a desk, no cloud-native path into NET exists. Any NET automation is workstation-bound, card-present and attended. Design for a human clicking submit, always.

- **Court sync is a commoditised, crowded market, do not build it.** At least five Israeli incumbents ship automatic download of pleadings, protocols, requests and decisions plus hearing dates into Outlook: Cligal (קליגל, ~20,000 Israeli lawyers, founded 2017, acquired 2022 by Axioma Information Solutions of the TASE-listed Top Group), Masig (משיג), Legal Office (ליגל), Yodfat (יודפת, operating since 1990), and Lawyal. Advise a client to buy one rather than competing with it.

- **None of those five vendors touches Legal Aid fee reporting.** Checked individually against their own marketing. Court sync is solved; the LAD fee-claim pipeline is served by nobody. That gap is where custom work is defensible for a legal-aid-appointed practice. Confirm with a sales call before pricing, since marketing pages are not full feature lists.

- **The Legal Aid fee schedule is STATUTORY, not tribal knowledge, do not pay to extract it from someone's head.** `תוספת שניה` (Appendix Two) to `תקנות הסיוע המשפטי`, implementing regulation 11, is a table of actions with fixed amounts organised by court type, procedural stage and party role. Amounts are CPI-indexed and republished annually by the Justice Ministry Director General in `רשומות` under regulation 11a, and judges may increase a fee by 50 to 100% in particularly complex cases. Sample District Court figures: לימוד עניינו של מבקש 1,253 NIS, דיון מוקדם 517, ישיבה ראשונה להוכחות 623; out-of-court items are lower, e.g. התייעצות רגילה 210. Bootstrap any classification logic from the regulation, then validate against real approved claims. Sources: [Wikisource](https://he.wikisource.org/wiki/%D7%AA%D7%A7%D7%A0%D7%95%D7%AA_%D7%94%D7%A1%D7%99%D7%95%D7%A2_%D7%94%D7%9E%D7%A9%D7%A4%D7%98%D7%99), [Nevo](https://www.nevo.co.il/law_html/law01/325_003.htm).

- **Watch for a mismatch between what a client says the tariff is and what the regulation says.** One client contact described two flat tiers (in the low hundreds of shekels); the statutory schedule is neither two-tiered nor those amounts. Could be in-office shorthand, approximate recall, or a different tariff entirely. A classifier built on the wrong assumption is worse than none: always reconcile the client's description against the published schedule and a real submitted claim before designing anything.

- **LAD claim data can usually be exported to Excel, which beats scraping the portal.** Where available this is the single highest-value artefact: it carries action type, claimed and approved amounts, status, dates and rejection reasons. It answers rejection rate, approval times and real tariff usage in one file, doubles as the labelled answer key for a classifier (an approved claim is by definition correctly classified), and serves as the daily reconciliation source. Ask for the export with maximum history before designing anything.

## Israeli law on cloud and client data (for law-firm clients)

- **Rule 19**, `כללי לשכת עורכי הדין (אתיקה מקצועית), תשמ"ו-1986`: a lawyer keeps secret everything a client brings to their knowledge, and must instruct staff on that duty and on information security.
- The Justice Minister set by regulation that "עורך דין ינקוט אמצעים הולמים לאבטחת המידע שיובא לידיעתו בידי לקוחו או מטעמו ואשר נשמר באמצעים דיגיטליים ובמערכות המידע בשליטתו ובשימושו", adequate measures, not named technology.
- **Offshore servers are expressly permitted**, conditionally: "ככל שהמשרד עושה שימוש בשירותים המאוחסנים על שרתים מחוץ לישראל (למשל אחסון בענן), תבוצע ההעברה בהתאם לדין, ותוך נקיטת אמצעים חוזיים או טכנולוגיים מתאימים להבטחת רמת הגנה נאותה."
- Cross-border transfer runs under `תקנות הגנת הפרטיות (העברת מידע אל מאגרי מידע שמחוץ לגבולות המדינה), תשס"א-2001`: destination protection no lower than Israeli law, with listed exceptions, plus a written undertaking from the recipient. In practice a signed DPA with EU standard contractual clauses satisfies this.
- **Practical conclusion: cloud is not the question, adequate measures are.** Israeli firms routinely work in the cloud, reaching client files and NET remotely; Cligal is cloud-based at roughly 20,000 lawyers. The strongest anchor in a client conversation is that they are almost certainly already on Microsoft 365, so the standard is one they have already accepted. Do check whether a government appointment contract imposes a data-location clause, since a state client can demand more than the law does.

## Conclusions / best practices

- **Buy the commodity, build the gap.** Recommend a practice-management vendor for court sync; build only the Legal Aid layer.
- **Never promise unattended NET or LAD submission.** Card-bound and card-present. Scope "prepare the packet, a human submits" and price any attended assist separately.
- **Lead discovery with the Excel export**, not a portal walkthrough. One file answers more than an hour of screen-share.
- **Terminology is ambiguous in the wild**, e.g. `כתב מינוי` vs `צו מינוי` for the appointment document. Build a Hebrew-English canonical glossary early; it doubles as the taxonomy any classifier runs on.
