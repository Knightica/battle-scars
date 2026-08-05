# Telegram

**Use for:** Invoice-filing bots, run-failure alerts on background-job projects, ad-hoc operational pings.
**Status:** Active

## Setup & access
- Bot token: `TELEGRAM_BOT_TOKEN`
- Invoice group chat ID: `INVOICE_GROUP_CHAT_ID` (negative -100... supergroup)
- Alert chat ID: `TELEGRAM_ALERT_CHAT_ID` (negative -100... supergroup)
- Bot must be a member of the target group before it can post or react
- Telegram Bot API base: `https://api.telegram.org`

## Scars & gotchas
- **Reactions fail in basic groups** - `setMessageReaction` only works in supergroups (chat_id prefixed -100...); basic groups silently reject it. Migrate a basic group to a supergroup before wiring reactions. The 👍 emoji is on Telegram's whitelisted standard-emoji list; custom/animated emoji require Telegram Premium and will fail for standard bots.
- **getUpdates with no offset pulls everything buffered (~24h)** - messages older than ~24h are dropped by Telegram and are not recoverable via polling. No dedup database is needed if your downstream storage already dedups: e.g. a Drive filename match keyed by `file_unique_id` is the dedup layer, so running the task twice on the same update window is a no-op.
- **file_unique_id is the right stable key, not file_id** - `file_id` can change across bot API versions. `file_unique_id` is stable for the lifetime of the file. Use it as a deterministic filename prefix (`${file_unique_id}__${original_name}`) so re-runs are idempotent and storage never accumulates duplicates.
- **Supergroup chat IDs are large negative numbers** - format is `-100XXXXXXXXXX`. Store as Number (not string) for comparison; string comparison can silently mismatch when the leading `-` is dropped or the ID is truncated. When a group is converted from a basic group to a supergroup its id changes shape entirely (short negative to `-100...`); auto-forwarding from the old group can mask the stale value, so verify the stored id after any migration.
- **sendTelegramAlert must be non-fatal** - if the alert itself errors (network blip, wrong chat ID), the parent task should log and continue. Never fail a business-critical job because alerting failed.
- **Reaction rate limit - ~20 per burst then ~30s cooldown** - Telegram enforces approximately 20 reactions per chat per burst; exceeding it returns a 429, which is non-fatal. Backfill pattern: ~300ms inter-message gap, send 20, wait for cooldown, send remainder. A clean implementation throws a typed `TelegramRateLimitError` carrying `retry_after`; the sender backs off via a checkpointed wait (zero compute cost) and retries.
- **Bot reactions - ✅/❌ are explicitly rejected** - the standard-emoji whitelist excludes all check-mark variants including ✅ and ❌. 👍 is the correct bot reaction for a success path, 👎 for failure. Code sending ✅ will receive a Bad Request from the Bot API.
- **Supergroup conversion changes the chat id** - a group that starts as a basic group and is converted to a supergroup mid-session receives a new id. Stored chat ids must be updated immediately after conversion; the old id stops receiving bot traffic.
- **Route task failures to the alert chat via an `onFailure` hook** - wire all background-task failures to a dedicated alerts supergroup via `onFailure` in your job config. The alert target must be a supergroup (not a basic group) for bot posting to work.
- **Bots can't read messages sent by other bots (getUpdates)** - a bot polling `getUpdates` does not receive messages posted by another bot in the same group. So you can't verify "did bot B's alert land?" from bot A's getUpdates; confirm via the sending platform's execution logs or a human eyeball. This bites any health-check bot that tries to auto-ticket another bot's error alerts.
- **HTML-format bot alerts with `parse_mode: HTML`** - for nice alerts (bold labels, monospace values, a copy-able code block), send with `parse_mode: HTML` and tags `<b>`, `<code>`, `<pre>`. Always HTML-escape any dynamic value (`&` to `&amp;`, `<` to `&lt;`, `>` to `&gt;`) or a stray char in an error message will make Telegram reject the whole send.
- **Capturing a group's chat id when the bot has Group Privacy ON (the default)** - a privacy-on bot in a group only receives messages that are commands addressed to it or that @mention it, so a plain "hi" posted in the group never reaches `getUpdates` and you cannot read the group's chat id from it. The poll comes back empty even though the bot is a member, which reads as "the bot isn't in the group". To onboard a bot to a new group: add the bot, then either post `/<cmd>@<botusername>` (an explicit mention always reaches it) or rely on the `my_chat_member` add event. Crucially, **`getUpdates` EXCLUDES `my_chat_member` by default** - pass `allowed_updates=["message","my_chat_member"]` or you poll an empty result. A fresh basic group returns a `-<digits>` id (not `-100...` yet); it changes on supergroup conversion (see above). Also check `getWebhookInfo` first: a webhook set on the bot drains `getUpdates` to empty.

## Conclusions / best practices
- Always use supergroups (not basic groups) as bot targets. Migrate before wiring reactions.
- Key dedup on `file_unique_id` not `file_id`. Filename pattern: `${file_unique_id}__${original_name}`.
- Store chat IDs as Number. Verify the -100... prefix is present after any group migration.
- Wrap all `sendTelegramAlert` calls in try/catch and swallow errors. Alert failure must never propagate.
- Run `getUpdates` with no offset to pull the full buffer; ack the offset after processing so the buffer shrinks each run.
- A bot only receives group document/photo messages via `getUpdates` if Group Privacy is OFF (BotFather -> Bot Settings -> Group Privacy -> Turn off) or the bot is a group admin. Also, bot-sent files are skipped by any `is_bot` filter, so end-to-end test files must come from a human account.

## Doc log
- **2026-07-07** - Added bot Group Privacy / is_bot-filter scar for group file collection.
- **2026-06-25** - Added bot-to-bot getUpdates invisibility + HTML alert formatting scars; re-confirmed supergroup-id-change behavior.
- **2026-06-25** - Initial consolidation.
