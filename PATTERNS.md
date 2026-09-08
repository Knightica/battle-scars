# Cross-platform patterns

The scars that show up no matter which platform you're wiring. Platform-agnostic rules distilled from many production integrations. When a lesson is specific to one platform it lives in that platform's file; when it keeps recurring everywhere, it lives here.

## Reliability

- **Idempotency is mandatory.** Every webhook handler and every broadcast must be safe to run twice. Use idempotency keys (on the source event ID, or a TTL'd key) so retries and redeploys never double-fire. A crisis redeploy (multiple versions shipped in minutes) only stays safe because of this.
- **Queue isolation: separate time-critical from bulk.** A big broadcast on the same queue as transactional sends causes head-of-line blocking (a ~700-message blast at 1 req/sec once delayed time-critical reminders by ~35 minutes). Give bulk sends their own queue and pacer, isolated from the critical-send queue (concurrencyLimit 1).
- **Soft-skip on crons, throw on webhooks.** Async crons should wrap third-party reads in try/catch and soft-skip on failure (no alert) because they run again. Webhook tasks should NOT swallow errors: let them throw so the source retries on its side.
- **Verify after async.** Silent failures are the most expensive kind. A task that returns "ok" while throwing inside a non-fatal catch (a missing prod env var, a missing link) breaks things invisibly. Add a post-run check ("did the thing actually complete?") or a watchdog cron for high-value flows. Throw on real errors, soft-skip only expected transient ones.

## Deploy & config

- **Sync env vars on every deploy.** Missing prod env vars are a top cause of silent failures. Push the local `.env` to prod automatically on each deploy (e.g. the Trigger.dev `syncEnvVars` build extension), scoped per project with an allowlist/prefix filter so prod never inherits another project's secrets.
- **One project per codebase.** Never let two codebases share one background-jobs project (e.g. a Trigger.dev project ref); worker overwrites make the other codebase's jobs vanish from prod.
- **Secrets in a keychain, never in repo or chat.** Long-term credentials go through a secret-manager wrapper, never committed, never pasted into a transcript. Rotate any key that lands in a transcript.
- **An MCP server's tool list is snapshotted when the session connects, so rebuilding a server mid-session leaves the agent blind to the new tools.** Add a tool to an MCP server, rebuild it, and a session that was already running keeps advertising the OLD tool list. The tool is on disk and fully working, but it is invisible: a tool-search step returns no match for it under every query, and calling it directly fails with "no such tool available." The trap is that this is indistinguishable from the tool never having existed, so the agent will confidently report the capability is missing and propose a worse fallback (one case: a draft-email tool had been added and built the week before, the running session predated it, and the agent concluded no draft tool existed and was about to send an email instead of drafting it). Never conclude a capability is absent from the tool registry alone. Grep the server's build output for the tool name before saying it does not exist, and compare the build file's mtime against the session start. Two fixes: restart the session so the server re-advertises, or invoke the built module directly with the same env the MCP config passes.

## Data & dedup

- **Dedup on a normalized key.** Normalize phones to digits-only `972...`, dedup contacts by email or a stable platform ID, and pick the right unit (dedup by CONTACT, not by lead, when one contact spans many funnels). Layer multiple checks when one signal is unreliable (iCalUID, then organizer email, then a time-window job guard).
- **Backfill atomically when adding a column.** New CRM column? Backfill all existing rows in one pass and verify zero failures, so old and new rows are consistent immediately.
- **One source of truth.** Render PDFs from the live URL with a query param (not a parallel template), compute folder names deterministically, store canonical IDs from the API (not dashboard row IDs). Avoid two representations that can drift.
- **Dedup the event, not the person - repeat purchases must surface.** A paid-signup pipeline that skips ALL side effects (sheet row, receipt, messaging) when the participant already exists treats every repeat purchase as a duplicate webhook. Real case: a repeat purchaser bought a second ticket under the same contact details - charged, invisible everywhere, found only by ledger reconciliation. Key event idempotency on the transaction id; gate only person-scoped effects (welcome messaging) on the person. Repeat person + new tx = new business that must hit the money trail.

## Email deliverability

- **SPF + DMARC before launching a sender domain.** DKIM alone is not enough. Without SPF/DMARC, legitimate confirmation emails land in spam.
- **Typo-guard email inputs.** A typo'd domain (`@gamil.com`, `@gmail.con`) is worse than a bounce: it may silently deliver to a typosquat. Add client-side typo guards at checkout plus server-side validation.

## Payments (cross-provider)

- **Never trust the webhook body alone.** Phone home to the provider's verify endpoint with the transaction UID to get authoritative status before acting. Ghost payments happen when you trust a sparse IPN. Most IPNs are unsigned, so the phone-home is your only real authentication.
- **Exactly one callback channel, never both.** A dashboard-configured callback AND a per-request callback both fire (0-1s apart) and double-issue invoices. Pick one channel and blank the other. This once produced 8 duplicate invoices before it was caught.
- **Fail-closed on duplicate-invoice guards.** Layer the guard (a fast local store plus the invoicer's own document list by tx UID). If both layers come back "unknown/error", refuse to issue and let the provider retry. Better a retried charge than a duplicate invoice.
- **Always ship a manual "mark paid" escape hatch.** Webhooks fail; give the admin a button so a stuck payment doesn't need a developer.

## Robustness

- **Per-call timeout on every external client.** All of them. A hung call with no timeout cascades into MAX_DURATION_EXCEEDED and the whole task does zero work (a messaging-platform lookup once hung, the task died after 5 minutes, and zero CRM writes happened). A 15s HTTP timeout fixed it.
- **Verify-by-HMAC on raw bytes.** Sign the exact JSON bytes on the client and verify those same received bytes on the server, no re-canonicalization, so client and server can never disagree on canonical form. Give each form/endpoint its own HMAC secret for clean rotation.

## Data hygiene

- **One canonical phone format everywhere: `972XXXXXXXXX`, digits only, no `+`.** Apply it across every system (CRM, messaging, ads, database). But never blind-prepend `972` to inputs that lack a `+`: non-Israeli contacts (US/UK/FR) get corrupted. Blank the unknown, don't guess.
- **Never blind-concat formatting onto a possibly-formatted value.** A node that did `"+{{phone}}"` on an already-`+972` value created 800+ `++` / `+` dirty records needing manual cleanup. Normalize, then format once.
- **Names that must be one language: transliterate before write.** If a system's names must be English, transliterate Hebrew input (with a char-by-char fallback) before any write. Decide the canonical language per system and enforce it at the boundary.

## Secret hygiene

- **Never source the whole `.env` into a shell.** Read single values via `grep`/`cut`. A debug handler that echoed a full service-account key JSON to stdout is how keys leak into transcripts. Rotate anything that lands in a transcript.

## UI / form

- **Hidden honeypot fields must use the clip pattern, not `left: -9999px`.** The off-screen offset inflates `scrollWidth` past 10k px; some mobile browsers auto-fit-to-content and zoom the whole page out to a thin strip. Use the sr-only `clip: rect(0,0,0,0)` pattern instead.
- **Headless Chrome enforces a ~500px minimum viewport; screenshots below that clip the right edge and mimic a real layout bug.** Requesting `--window-size=430,...` still lays the page out at ~500px, so a 430-wide screenshot canvas crops the right ~70px, and right-aligned elements look cut off even though the page has zero overflow. Before "fixing" a phantom right-edge overflow, verify by instrumenting the live page (`document.documentElement.clientWidth` vs `scrollWidth`) or screenshot at ≥500px. Real phones render fluid down to 320px fine.

- **Canvas input (signature pads etc.): never fix the bitmap size while CSS scales the element.** A canvas with hardcoded `width`/`height` attributes plus `max-width: 100%` renders scaled down on mobile, but pointer coordinates map into the unscaled bitmap: strokes land offset from the finger. With a library like react-signature-canvas, omit `width`/`height` from its canvas props so the library sizes the bitmap from the rendered element (devicePixelRatio-aware); it skips that logic entirely when both are fixed. Also set `touch-action: none` on the canvas so the page doesn't scroll mid-stroke.

## Rendering, print & RTL

- **Chrome `page.pdf` misplaces `position:absolute` + `transform` elements across page breaks.** A deck that looked right on screen printed its vertically-centred diagrams (`top:50%; translateY(-50%)`) hundreds of px too low and clipped. Inject a print stylesheet that puts those elements back in normal flow (`position:static; transform:none; margin:auto 0`) before calling `page.pdf`; `@page { size: 1440px 900px; margin:0 }` plus one `section` per page with `break-after:page` gives a clean one-slide-per-page vector PDF with fonts embedded and links live.

- **Playwright runs on the installed Google Chrome with `channel:'chrome'`, no browser download.** `npm i playwright` in a scratch folder is enough; `npx playwright` alone resolves the package but leaves `import 'playwright'` unresolved. Headless Chrome's `--screenshot` CLI flag captures only the first viewport, useless for scroll-snap decks; drive it from Playwright and `scrollIntoView` each section.

- **Pillow's PDF writer needs a JPEG encoder that some Python builds lack (`KeyError: 'JPEG'`).** Save with `format="PDF"` and no `quality` for a lossless image PDF, or skip Pillow and print from Chrome, which also gives vector text.

- **Inline SVG in an RTL document: set `direction:ltr` on the `<svg>` and `rtl` only on Hebrew `<text>` runs.** With the page's `rtl` inherited, `text-anchor:start/end` flip, so right-anchored lane labels run off the canvas and corner tags spill outside their boxes. Numbers inside Hebrew runs still order correctly once the text element itself is `rtl`.

- **A local HTML file with Hebrew needs `<meta charset="utf-8">` in the first bytes.** Some sandboxed artifact viewers add it automatically, but a plain `python3 -m http.server` or a `file://` open does not, and Chrome guesses Latin-1 and renders mojibake. Put it before `<title>`.

- **Gmail strips `dir` from `<html>` and `<body>`; every table cell and paragraph in an RTL email needs its own `dir="rtl"`.** With only `text-align:right`, the paragraph is laid out LTR and right-aligned, so periods and question marks land at the start of each line. Put `dir="rtl"` (and `direction:rtl` inline) on each `<td>` and `<p>`, and isolate Latin runs and numbers with `<span dir="ltr" style="unicode-bidi:isolate">`.

- **Email images: Gmail drops `data:` URIs and SVG, and no client loads custom fonts.** Host every image as a raster PNG or JPEG on a public https URL and design for the fallback stack (Arial, Georgia, Courier New); brand fonts never render in an inbox. Do not push base64 images through a send tool either: a few-hundred-KB body has to travel through the tool call and Gmail discards it anyway.

## SEO & GEO

- **AI answer-engine crawlers (GPTBot, ClaudeBot, PerplexityBot) generally don't execute JavaScript.** Content that exists only inside a hidden div injected into a modal on click (e.g. a "view more" overlay) is invisible to them even though it's technically in the raw HTML. Googlebot renders JS and copes fine, but for real GEO / AI-citability, give each meaningful content unit (case study, article, product) its own server-rendered, crawlable URL, not just a client-side reveal. Keep the JS-driven UX for real visitors (intercept the link click, preventDefault, show the animated version) while the plain `<a href>` still resolves to a real page underneath.

## Monitoring & alerting

- **Never use a consumed Telegram `getUpdates` feed as the source of truth for alerts.** Any other reader of the same bot token drains the offset window, and alerts vanish before a scan sees them (this bit us twice: a critical scheduled job's 429 missed entirely, and a batch of automation failures found only via a screenshot). Health checks must query the failing systems directly (Trigger.dev runs with `list_runs status=FAILED/CRASHED`, n8n error executions with `n8n_executions status=error`, Netlify deploy states) and treat Telegram only as a cross-check that the alert pipeline itself works.

- **"0 processed" is ambiguous: it means either "nothing to do" or "my input is dead," and the two look identical.** A poller over an external inbox (a Drive folder, an inbox, a webhook queue) reports the same healthy zero whether the source is quiet or the source stopped reaching it. One lesson-delivery poller ran clean for two weeks while every incoming file landed in a different folder than the one it watched; the failed-run scan, the queue scan, and the deploy state were all green throughout. Monitor the expected input, not just your own failures: if work was expected and nothing arrived by the deadline, that is the alert. Applies to any "absence of work" signal: a silent queue, an empty sync, a webhook that stopped firing.

- **An auth failure inside a monitoring scan is a finding, never a skip.** An API key returned 401 on many consecutive health checks over several weeks, and every report still read clean, so a long stretch of real errors was invisible. A scan that cannot authenticate must say "`<system>` unmonitored" in the report header and open a ticket for the credential itself; it never contributes to an all-clear.

- **An in-memory flag is not a dedupe, it's a per-pageload dedupe.** A client-side event that should alert once per visitor (proposal opened, demo started, trial activated) guarded by a plain `var fired = false` re-fires on every reload, because the variable dies with the page. The result is a stream of real-but-duplicate alerts that look exactly like genuine traffic, and the people receiving them start distrusting the channel. Guard anything that pages a human with `sessionStorage` (per visit) or `localStorage` (per browser), and keep the guard on the notification only, never on the UI behaviour it accompanies. The tell is an event that fires far more often than the page-load event it should be bounded by.

- **Guard your own headless renderer at the source, not with a user-agent filter.** Any page that your own automation visits (a Puppeteer PDF render, a screenshot service, a scraper, a synthetic uptime check) will fire that page's analytics and alert hooks. Filtering server-side on `HeadlessChrome` in the UA works right up until someone adds a `setUserAgent` call for an unrelated reason, and then it fails silently and forever, with no error and no way to tell the phantom events from real ones after the fact. Have the renderer visit a distinguishable URL (`?mode=pdf`, a header, a query flag) and make the page skip tracking on that signal. Keep the UA filter too; two independent guards, because this failure mode is invisible.

## Session workflow

- **If your workflow auto-generates an end-of-session commit (a changelog or log-file entry), make that commit before opening the PR, not after.** On a repo where direct pushes to main are forbidden and every change must go through a branch + PR, writing that log commit after the PR is already open forces a bad choice: either push directly to main (breaking the rule) or open a second PR for one file. Order it: finish the work, commit the log entry on the feature branch, then open the PR.

## Consent gate at the send chokepoint

Every outbound-messaging codebase gets ONE function all sends pass through, and that function REQUIRES a consent declaration in its signature: `{ kind: "transactional" }` (direct consequence of a user action, always sends) or `{ kind: "marketing", contactId }` (checked against the stored opt-out at send time, DENY BY DEFAULT when consent is unprovable). Making the parameter compile-required means no future code path can send without classifying itself; storing an opt-out flag that nothing reads is worse than useless because it looks like protection. Born from a nurture-resend incident: an unsubscribe button wrote a flag for two months that no send path ever read.
