# Supabase

**Use for:** Postgres database, RLS-gated auth, private file storage (signed URLs), and server-side type generation.
**Status:** Active

**TL;DR:** RLS is **deny-all by default** - open only what's needed; an `using(true)` policy is an anti-pattern. For single-admin apps, RLS on `authenticated` is enough. Keep **all writes server-side via the service-role key** and don't ship `VITE_SUPABASE_*` to the browser. Run DDL via the **Management API** (with a `User-Agent` header) - the MCP is unauthenticated for DDL. **Regenerate TS types after every migration.** Private buckets + short-TTL signed URLs for any PII/contract files.

## Setup & access

- Access token lives in env as `SUPABASE_ACCESS_TOKEN`; service-role key is `SUPABASE_SERVICE_ROLE_KEY`.
- No local CLI needed - the Management API covers migrations and type generation.
- Regenerate TS types after every migration: `npx supabase gen types typescript --project-id <ref>`. Stale types cause silent runtime mismatches.
- For single-admin apps, skip separate admin roles. RLS on `authenticated` alone is enough (no `admin` role needed).

## Scars & gotchas

- **Self-service DDL + key retrieval via the Management API (PAT in a secret store)** - The Supabase MCP is unauthenticated for DDL, but a Personal Access Token can live in a secret store (e.g. the OS keychain). With it you can run migrations (`POST /v1/projects/{ref}/database/query` with a `User-Agent` header) AND fetch the service-role key (`GET /v1/projects/{ref}/api-keys?reveal=true`, filter `.name=="service_role"`) entirely yourself, no human paste. Write the fetched key straight into the target `.env`/config without echoing it to the terminal (only its length) so it never lands in a transcript.

- **Org membership is account-wide, not per-project** - Adding a member to a Supabase org grants access to *every* project in that org; there is no per-project member scoping on Free/Pro plans (project-scoped roles are Team/Enterprise only). Inviting an external developer to an org that holds unrelated projects would expose all of those databases to them. Keep unrelated projects in isolated orgs, or skip the dashboard invite and hand the dev just the service-role key + connection string for the one project they need.

- **Cloudflare 1010 blocks Management API calls without a User-Agent** - Run migrations via `POST api.supabase.com/v1/projects/{ref}/database/query` and include a `User-Agent` header. Without it, Cloudflare returns a 1010 error.

- **MCP secrets with "all" context fail silently** - Setting Supabase secrets with the "all" context via MCP produces no error but does not write. Set each secret per-context (prod, preview) explicitly via the raw API or MCP with a specific context.

- **node-22 required for @supabase/supabase-js Realtime** - The Realtime client depends on native WebSocket. Node 21 throws "Node.js 21 detected without native WebSocket". Pin `runtime: "node-22"` in trigger.config.ts whenever Supabase Realtime is in the dependency tree.

- **Compensating transactions on multi-step server actions** - If a later step (e.g., PDF generation, third-party call) throws, roll back all prior inserts (parent row + child rows) in the same action. A partial insert with no cleanup leaves orphaned data that is hard to reconcile.

- **Private buckets + short-TTL signed URLs for sensitive files** - Use private storage buckets for PDFs and contracts, never public. Serve via short-TTL signed URLs. Supabase Storage is the system of record; any Drive/email copies serve archival and delivery respectively.

- **Reuse tables with boolean flags instead of parallel tables** - When a new entity type (e.g., temporary groups) overlaps heavily with an existing one, add an `is_temporary` flag + nullable fields + a join table rather than a separate table. A parallel table breaks all existing queries that reference the original. Downstream code stays unchanged.

- **Validate enum columns with Zod before insertion** - Supabase enum columns (including Hebrew-valued status fields) are not validated at the client layer. Use a Zod schema to reject invalid values before hitting the DB.

- **Supabase MCP is unauthenticated for DDL** - `execute_sql` and `apply_migration` both fail Unauthorized via the MCP when no `SUPABASE_ACCESS_TOKEN` is configured. A runtime that only carries `SUPABASE_URL` + `PUBLISHABLE_KEY` + `SERVICE_ROLE_KEY` cannot run DDL: service-role authenticates against PostgREST, not raw Postgres, so DDL is impossible without the DB password or a PAT. Run migrations via the Management API, Studio SQL editor, or configure a PAT in the MCP server.

- **Service-role-only RLS on server-side persistence tables** - Tables used for server-side-only persistence (e.g., message logs, confirmation state, nurture queues) should carry service-role-only RLS - no anon or authenticated policy. Keep a dedicated blob/store per workflow for idempotency rather than reusing polling stores; keeps concerns clean and gives an isolated dashboard view for monitoring.

- **Recursive RLS lookups block login on admin accounts** - RLS policies that perform recursive table lookups (e.g., checking a staff hierarchy table from within a policy on that same table) can deadlock admin login at auth time. Fix: replace inline recursive checks with SECURITY DEFINER helper functions and add explicit GRANT statements on the affected tables so the helper can read them without triggering the policy guard.

- **With RLS on, do NOT expose Supabase client env vars to the browser** - When all write operations go server-side via service_role, intentionally omit `VITE_SUPABASE_*` env vars from the site. Any client-side insert will fail RLS and surface the intent to bypass it. The frontend has no direct Supabase access; that is the correct security posture.

- **Start RLS as deny-all, open only what is needed** - Default stance on every new table is deny-all (no policy = no access). An open `using(true)` policy as seen in reference apps is an anti-pattern: easy to copy, hard to audit, and catches engineers off-guard when a later table inherits the same template. Enable only the minimum required access and document why each policy exists.

- **Org consolidation can silently delete auto-created projects** - When a Supabase org is renamed or removed, any project auto-created inside the dissolved org vanishes. Recreate critical projects explicitly under the surviving org and confirm they exist before decommissioning the old one. Do not rely on auto-creation for anything with live data.

- **Storage `createSignedUrl` verifies the object exists** - unlike a raw S3 presign, Supabase Storage returns "Object not found" when signing a URL for a nonexistent path. You cannot create a signed URL ahead of the upload, and a DB row pointing at a missing storage object will fail at sign time, not at fetch time.

- **Service-role-only tables still need RLS explicitly enabled (`alter table ... enable row level security`)** - Tables created for server-side-only use ship with RLS *disabled* unless the migration enables it; the service-role client works identically either way, so nothing in testing catches the omission. Supabase's security scanner does: it emails a critical `rls_disabled_in_public` alert ("anyone with your project URL can read, edit, and delete all data"), because the anon key reaches the table via PostgREST. The fix is a zero-risk one-liner per table - enable RLS with zero policies; service role bypasses RLS by design, so backend jobs are unaffected and outside access is fully blocked. Two server-side-only tables once lived a month with RLS off despite the documented service-role-only pattern above - the pattern only holds if the enable statement is actually in the migration. Cross-check with the security advisors (`get_advisors(type=security)`); the target end state is the INFO-level "RLS enabled, no policy" lint.

- **A new table with RLS-on + no policy fails READS silently (empty result, no error)** - `apply_migration` can create a table that comes up with RLS enabled and zero policies. Writes fail loudly, but a `SELECT` through the `authenticated` client just returns **zero rows with no error**. An app that generates data from that table renders **empty** with nothing in the logs - it reads as "the data got deleted" when the data is fully intact. Lesson: when a migration adds a table the app will READ from, create the `authenticated`-select policy **in the same migration**, and after any new-table migration do a live authenticated read (not just `tsc`/`build`, which can't see RLS). Grepping `pg_policies` for the new table (policies count = 0 while `relrowsecurity` = true) pinpoints it instantly.

## Conclusions / best practices

- Always run `npx supabase gen types typescript --project-id <ref>` immediately after applying a migration. Never skip it.
- Always include a `User-Agent` header on Management API calls.
- Use private buckets by default for any file with PII or contractual content. Public buckets are opt-in only.
- RLS on `authenticated` is the only access gate needed for single-admin internal apps.
- Compensating-transaction pattern: wrap multi-step server actions so any failure after the first insert rolls everything back.
- Prefer table reuse with a boolean discriminator over parallel tables unless the schemas diverge significantly.
