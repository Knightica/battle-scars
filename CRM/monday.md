# Monday.com

**Use for:** CRM board read/write, lead and contact management, status updates, column value mutations, scheduled-task tracking
**Status:** Active
**Last validated:** 2026-05-24

## Setup & access

- Monday API tokens are account-scoped. A token only sees the boards in the workspace it belongs to. To reach a separate Monday account (e.g. a client's own workspace), wire a separate API token and a dedicated MCP instance keyed to that account's token.
- Recommended CRM pattern: separate Contacts board + Leads board. Do not merge them into one.
- If you store phone numbers, pick one canonical format and normalize on write. A common choice is digits-only country-code text (e.g. `972...`, no `+`). All lookups must normalize to the same format before calling `items_page_by_column_values`.
- `board_relation` columns are silently ignored in `create_item` mutations. Set them via a separate `change_column_value` call after the item is created.
- Mirror columns return `text: null, value: null` in GraphQL. The real value lives only in `display_value`. Add `... on MirrorValue { display_value }` to your fragment and fall back to it in the mapper.

## Scars & gotchas

- **Checkbox columns don't filter on `[true]`** - Querying `items_page(query_params: {rules: [{column_id, compare_value: [true]}]})` against a boolean/checkbox column returns **zero items** even when rows are checked, no error, just a silent empty set (this once produced a false "0 conversions" scare mid-event). The checked value is stored as `{"checked":true}` and the display `text` is `"v"`. Reliable path: don't server-side filter checkboxes, page the board and filter client-side on `column_values.value` containing `"true"` (or match `text == "v"`). Reading a single row's checkbox is fine; it's the `query_params` rule that misbehaves.

- **Per-field rate limit surge** - When many child runs hit a single-column mutation (e.g. a status setter) concurrently, Monday's per-field rate limit fires. The 429 arrives as HTTP 200 with `extensions.retry_in_seconds` in the body, not as a real HTTP 429. Runs sleep inside the `gql()` retry loop honoring that field; wall times balloon to 1-3 min while compute stays at ~290-416ms (resilience working, but expensive). Fix: add a queue with `concurrencyLimit: 2-3`, replace any `setTimeout` in retry helpers with `wait.for`, and debounce by a stable per-entity key with a 5s window.

- **Cron collision FIELD_LIMIT_EXCEEDED** - Two crons firing at the exact same minute (e.g., `0 * * * *` and `*/15 * * * *` both at :00) can hit Monday's per-field `items_page` concurrency limit and surface `FIELD_LIMIT_EXCEEDED`. Fix: stagger crons so they never share a fire minute (e.g., use `:02/:17/:32/:47` instead of `*/15`). Also bump `maxAttempts` on the parent.

- **Token scoping (multi-account)** - Monday API tokens are account-scoped. A token for one account cannot see another account's boards. For any multi-account engagement, run a dedicated MCP instance per account. One session can hold both MCP instances simultaneously.

- **Soft-skip rule (crons vs webhooks)** - Async cron tasks must wrap their top-level Monday read in `try/catch` and return `{status: "skipped", reason: "monday_unavailable"}` on Monday API errors. This suppresses `onFailure` alert spam during outages (a ~1.5h API outage once triggered ~85 alerts). Webhook-triggered tasks must NOT be wrapped, because upstream systems (n8n, invoicing, chat platforms) will retry on failure. Monday is optional for crons; it is critical for immediate webhook flows.

- **Contacts vs Leads dedup** - Dedup by Contact, not by Lead. One Contact can have many Leads across funnels. A webinar-style signup adds a note only when a duplicate Contact is found (a separate attendance board can track repeats). A direct funnel may always create a new Lead even when the Contact already exists, because each funnel entry is independently tracked. (Note: the exact policy is business-specific, see the 1-Contact-1-Lead variant below.)

- **Phone normalization for lookups** - Normalize all phones to a single canonical format (e.g. digits-only `972...`) before any `items_page_by_column_values` search. Monday's phone column stores text exactly as written; raw phone strings with `+` or spaces will not match if the stored value has none.

- **Backfill new columns atomically** - When a new column is added to a live board (e.g., a label column), backfill all existing rows in one script run via the Monday MCP. At a few hundred rows the MCP completes in ~1 min. Verify 0 failures before treating the column as canonical.

- **board_relation silently ignored on create_item** - Passing a `board_relation` column value inside `create_item` does nothing and returns no error. Always issue a separate `change_column_value` mutation after the item is created to set the relation.

- **Mirror column text null** - Columns of type Mirror always return `text: null` and `value: null` in standard GraphQL column_values fragments. The real value is only in `display_value` on the `MirrorValue` union type. Add `... on MirrorValue { display_value }` to every column_values query that may touch mirrors, and fall back to `display_value` in the mapper. Missing this once caused a ~150-recipient messaging batch to fire with zero messages sent.

- **Status column wants label id, not index** - `{index: N}` in a status mutation is wrong. Monday expects `{value: {id: "<uuid>"}}` using the label's actual UUID id, not its positional index. Passing the index silently produces incorrect status or a no-op. Retrieve label ids from the board's column settings query before building mutation payloads.

- **items_page_by_column_values ignores board_relation filters** - Querying `items_page_by_column_values` with a board_relation column as the filter silently ignores the filter (or throws) and does not return expected results. Workaround: fetch all items on the board and match in-memory using `linked_items` from the `BoardRelationValue` fragment (`linkedPulseIds`).

- **1 Contact = 1 Lead is a valid alternative policy (business-specific)** - Some CRMs enforce exactly one Lead per Contact across all time. Re-engagement then updates the existing Lead in place: reset status, refresh `date`/`source`/`ad_name`, audit-comment the prior state. When 2+ Leads exist for one Contact, an upsert routine consolidates by a status-rank ordering, protecting won/high-value statuses from downgrade. This directly contradicts the "direct-funnel always creates a new Lead" rule above. Both rules are correct for their respective businesses. Apply per-business, do not generalise.

- **Dedup tiers** - A robust Contact lookup order: phone exact match -> email match -> fuzzy phone (Levenshtein <=1 on last 9 digits) -> create new Contact. The fuzzy tier catches common typo variants without false-positives.

- **Contact names may need to be single-script** - Some boards standardize Contact names to one script (e.g. English only). If input arrives in another script (e.g. Hebrew), transliterate before any `create_item` or `change_column_value` write. Decide the policy once and enforce it on every write path.

- **Rotate any token that touches a transcript** - If a Monday API token is ever exposed (e.g. pasted into a chat transcript), rotate it before further production use. First audit every n8n credential and Trigger.dev env var that references it: a blanket rotation without an audit will silently break workflows that use the old value.

- **Recovering a deleted item: API `undo`/`batch_undo` is 403 for service accounts** - `undo_action` (`batch_undo` mutation) returns `USER_UNAUTHORIZED` for the API/service-account user, so you can't programmatically restore a deleted pulse. Two recovery paths: (1) **Reconstruct it** - `get_board_activity(includeData=true)` returns the `create_pulse` entry with the full `column_values_json` (and the `delete_pulse` entry with `action_record_uuid`), so you can recreate the item exactly (new id); (2) **Recycle Bin** - deleted items sit in Monday's UI Recycle Bin and restore with their **original** ids. Don't do both (dupes).

- **Set status/dropdown by `{label:"Text"}`, not `{index:N}`** - in raw GraphQL `create_item`/`change_column_value`, `{index:N}` uses the label's positional index, which often does NOT equal the label id you hold in a constant (a label's id and its index frequently diverge, e.g. id 5 sitting at index 0). Passing an id as an index silently sets the wrong label. Use `{"label":"Follow Up"}`, which is unambiguous. Checkbox = `{"checked":"true"}`; link column = `{"url":"…","text":"display text"}`.

- **NEVER delete a CRM item during testing unless verified as a test by EXACT phone/id** - a live production CRM receives real inbound leads at any time; a lead created during a test session may be a genuine customer, not your test. In one incident a real inbound lead (a different number than the tester's) was deleted by inferring "that's probably the test" from timing alone. Only delete items whose phone matches the exact number being tested, or ids you explicitly created as tests. When unsure, ask.

## Conclusions / best practices

- Always normalize phones to one canonical format (e.g. `972...` digits-only) before any CRM lookup.
- Use a shared `gql()` wrapper that inspects response bodies for `extensions.retry_in_seconds` and sleeps accordingly. Standard HTTP-status-based retry misses Monday's field-concurrency throttling.
- Keep Contacts and Leads on separate boards. Dedup at the Contact level; decide per-business whether multiple Leads per Contact are allowed.
- Cron tasks: soft-skip on Monday errors (no alert). Webhook tasks: throw on Monday errors (let upstream retry).
- Stagger all crons so no two fire at the same minute on the same board.
- For any mutation to a `board_relation` column, always issue it as a second `change_column_value` call after item creation, never inside `create_item`.
- When adding a new column, backfill existing rows in one atomic script before relying on the column downstream.
- Multi-account Monday access requires a separate token and a separate MCP instance per account.

## Doc log

- **2026-06-25** - Initial consolidation.
- **2026-07-01** - Added deleted-item recovery (undo 403 / activity-log reconstruct / Recycle Bin), label-vs-index status setting, and the "never delete unverified CRM items" scar.
- **2026-07-10** - Added checkbox/boolean `query_params` filter scar (`compare_value: [true]` silently returns 0; filter client-side on `value` instead).
