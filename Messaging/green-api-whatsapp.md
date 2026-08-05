# Green API (WhatsApp)

**Use for:** WhatsApp group creation and member management via Green API - provisioning groups, syncing membership, verifying join success.
**Status:** Active

## Setup & access
- Instance ID: `GREEN_API_ID_INSTANCE`
- Token: `GREEN_API_TOKEN`
- Base URL: `GREEN_API_BASE_URL` (optional; defaults to `https://api.green-api.com`)
- Method URL pattern: `${BASE}/waInstance${id}/${method}/${token}`
- All calls are POST with JSON body and `content-type: application/json`

## Scars & gotchas
- **Chat ID format is `{phone}@c.us`** - phone must be in `972XXXXXXXXX` format: country code + digits, no leading `0`, no `+`. A phone stored with a leading `0` or `+` will fail silently or reach the wrong subscriber. Enforce the format with a `chatIdForPhone()` helper.
- **createGroup returns an optional inviteLink, not a required one** - verify success by checking `res.created === true` AND `res.chatId` is present, not by checking `groupInviteLink`. The invite link may be null even on a successful create.
- **Silent add/remove failures** - `addGroupParticipant` and `removeGroupParticipant` can return HTTP 200 without actually changing membership (e.g. phone not on WhatsApp, number has a bad prefix). Do not trust the API response alone. After provisioning, call `getGroupParticipants` (via `getGroupData`) to read the actual member list, diff against expected members, and alert on anyone who did not join. Include the group invite link in the alert so they can join manually.
- **Region-specific hosts** - Green API may route some instances to region-specific endpoints. If calls fail with connection errors, set `GREEN_API_BASE_URL` to the instance-specific host shown in the Green API dashboard.
- **Bulk-adding members trips Meta's spam guard (`423 account_reachout_restricted`).** Creating a WhatsApp group and adding many members at once (or a burst of `addGroupParticipant` calls on a fresh group) gets the instance temporarily blocked from adding participants: Green API returns HTTP **423** with `"account_reachout_restricted to timestamp: <unix>"`, and the block persists until that timestamp (~24-48h). A retrying task then re-fails, spamming the alert channel. **Workaround: never bulk-add. Seed the group with ONE trusted admin number only (the group owner), send them the invite link, and let them add members by hand.** The app keeps the `chatId` for reminders / file delivery but never manages the roster via API. Removals (`removeGroupParticipant`) do NOT trip the guard - only adds/reach-outs do.
- **Sending text:** the `sendMessage` method (`{chatId, message}` returns `idMessage`) sends into a group by its `@g.us` chatId. WhatsApp markup works: `*bold*`. Verify a call was accepted via `res.idMessage` presence (but see the next scar - accepted is not delivered).
- **`sendMessage` returns `idMessage` (looks like success) even when the instance is NOT a member of the target group** - so a message to a stale/wrong `@g.us` id silently vanishes: no error, no `sendErrors`, nothing delivered. This can mask a long-running reminder outage when the stored `chatId` points at a group the bot was never added to (e.g. a manually-created parallel group instead of the app-provisioned one). `res.idMessage` presence is NOT proof of delivery. Guardrails: (1) only ever use group ids the automation itself created - a hand-made group means the app messages a dead chat; (2) to actually verify deliverability, call `getGroupData(chatId)` and confirm the instance number is in `participants`; a bare `idMessage` is not enough.

## Conclusions / best practices
- Always normalize phones to `972XXXXXXXXX` before any Green API call. Strip leading `0` or `+`.
- Never assume `createGroup` succeeded from HTTP 200 alone. Check `created` + `chatId` fields explicitly.
- Always run a post-provisioning verification step: fetch actual participants, diff vs. expected, alert on gaps with an invite link.
- Keep `GREEN_API_BASE_URL` as an env var override so instance migrations don't require a code deploy.
- Wrap every Green API call in retry logic (transient 5xx errors are common); a task-level retry config covers this.

## Doc log
- **2026-07-23** - Added the silent-send scar: `sendMessage` returns `idMessage` even when the instance isn't in the target group, so a stale/wrong `chatId` drops messages with no error. Verify deliverability via `getGroupData` participants, not `idMessage`.
- **2026-07-13** - Added the `423 account_reachout_restricted` spam-guard scar + admin-seed workaround (switch group provisioning to seed only the owner's number and let them add members manually). Added the `sendMessage` text-send note.
- **2026-06-25** - Initial consolidation.
