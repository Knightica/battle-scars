# Lovable

**Use for:** Nothing we build ourselves. It's an AI site builder that clients use to design pages, which then get ported into a real repo and hosted separately.
**Status:** Active (client-side tool)

**TL;DR:** A Lovable export is a normal Vite + React + TypeScript + Tailwind + shadcn/ui project wrapped in editor scaffolding, plus a layer of plausible-looking code that is quietly wrong. The scaffolding is obvious and easy to strip. The wrong code is the dangerous part: hardcoded hostnames, invented date logic, and stale links that look intentional. Budget time for a copy-and-constants audit, not just a dependency cleanup.

## Setup & access

- Strip on the way in, and re-check after any future re-sync from the client:
  - the `lovable-tagger` dev dependency and the `componentTagger()` plugin in `vite.config.ts`
  - the Lovable script tag in `index.html` (`<script src="https://cdn.gpteng.co/gptengineer.js" type="module">`), often guarded by a "DO NOT REMOVE THIS SCRIPT TAG" comment. Remove it anyway.
  - the `.lovable/` planning directory (worth reading before deleting, see below)
  - `public/lovable-uploads/`, a UUID-named dumping ground. On one export: dozens of images, tens of MB, only a handful actually referenced, and some of those were byte-identical duplicates of the same logo.
  - `twitter:site` pointing at `@lovable_dev`
  - `bun.lock` / `bun.lockb` shipped alongside `package-lock.json`
  - a README that is Lovable's own onboarding doc, not the project's
- Also needs judgment, not just deletion: every hardcoded URL and hostname, any date or schedule logic, `<html lang="en">` left over on a right-to-left site, and duplicate data stores (see the Supabase scar below).

## Scars & gotchas

- **Exports carry hardcoded hostnames that may already be dead** - One export inlined a webhook URL as a constant, pointing at a subdomain that had gone NXDOMAIN months earlier after a domain migration. Every form submission on the exported page would have failed with a generic client-side error. The path was correct; only the host was stale. Grep every export for absolute URLs and resolve each one before shipping.

- **It invents schedule logic instead of taking a configured date** - Rather than accepting the event date as input, an export computed "the next Sunday at a fixed time" in the browser, seeded from a first-event date months in the past. The actual event fell on a different weekday, so the hero, the countdown, and the add-to-calendar link would all have advertised the wrong day to every visitor. Replace derived-schedule heuristics with one explicit configured value, sourced from whatever the backend actually fires against.

- **Fixing the date logic is only half the fix: the weekday can also be hardcoded into the copy** - After replacing a "next Sunday" heuristic, a form heading still read a static weekday name in the local language, sitting directly above a now-correct date and countdown. Same file, different failure mode. After correcting any date logic, grep the copy for weekday names, month names, and anything else that restates the date in prose, and derive it (`Intl.DateTimeFormat`) instead.

- **Stale links get duplicated across files, including into dead code** - A meeting link from an old event was hardcoded in three places: a thank-you page's calendar action, and twice more inside a component that was imported nowhere. Deleting the dead component removed two of the three; a repo-wide grep found the last one. Grep for the value, do not trust the component tree, and delete unimported components rather than leaving them to be "fixed" later.

- **Lovable wires its own Supabase project and ships the key in the bundle** - An export wrote every form submission a second time into a separate Supabase project as a "best-effort backup," with the publishable key compiled straight into the JS. Nothing ever read from it. That's a second, unmanaged copy of customer PII, guarded by a public key, sitting outside the system of record. Strip it unless someone can name what reads from it. Watch for **two** separate Supabase client modules in the same export (a hand-written one and an auto-generated "do not edit" one) reading different env var names.

- **`.lovable/plan.md` is the best documentation in the export - read it before you delete it** - It records the prompt-level intent behind the last change: the exact webhook URL, the payload contract, the field-handling rules. On one export it confirmed a hostname had been intentional-at-the-time rather than a typo, and it spelled out the field-by-field payload the receiving webhook expected. Read it, fold anything worth keeping into the repo README, then delete the directory.

## Conclusions / best practices

- **Port it into a repo you control, do not host from Lovable.** One repo, one hosting site, a subdomain on DNS you manage. The client keeps designing in Lovable and hands over exports.
- **Document the port in the repo README.** A "Ported from Lovable" section listing exactly what was removed and why, so the next re-sync does not quietly reintroduce all of it. The client will keep editing in Lovable, so this happens more than once.
- **Diff against your own file, not between the client's exports.** Their commit history double-counts changes already ported.
- **Verify against the built bundle, not the source.** After the port, grep the build output for the old hostnames, stale links, and removed SDKs. Cheap, and it catches anything a stale import left behind.
- **Assume the videos are enormous.** One export carried tens of MB of testimonial video imported through the bundler as assets. It builds and deploys fine, but every clone and every deploy pays for it. Flag it, move to a CDN when there's time.
