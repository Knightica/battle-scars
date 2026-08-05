# Trigger.dev

**Use for:** Background jobs, scheduled crons, AI workflows, webhook receivers, long-running tasks with waits. A strong default backend for anything code-heavy.
**Status:** Active

**TL;DR:** **One Trigger project per codebase**, sharing a project ref silently kills the other codebase's crons (worker overwrite). Use `@trigger.dev/sdk` v4 (never v2 `client.defineJob`), pin **`node-22`**, and put **`syncEnvVars` with a per-project prefix allowlist** on every deploy, or prod runs on missing or leaked secrets. Crons must **throw on real errors** (soft-skip only known transients) and guard against silent 100%-no-op. Idempotency keys everywhere (30d TTL); blasts get their own queue, never shared with time-critical sends.

## Setup & access

- One org holds all projects. One Trigger project per codebase. Never share one project ref across two repos.
- Project refs have the shape `proj_xxxxxxxxxxxxxxxxxxxx`. Keep a registry mapping each codebase to its own ref.
- SDK: `@trigger.dev/sdk` v4. Never `client.defineJob` (v2, breaks everything).
- Runtime: pin `node-22` in `trigger.config.ts`. Node 21 breaks `@supabase/supabase-js` Realtime.
- Declarative cron (`cron:` inside `schedules.task`) fires only for tasks present in the LATEST deployment.
- Env vars: use `syncEnvVars` build extension with an explicit prefix allowlist. Scope the allowlist per project so prod never inherits another project's secrets.
- Failure alerts: `onFailure` hook in `trigger.config.ts` posts to a dedicated Telegram bot/group. Use a per-project alerts bot, not a shared conversational bot.
- The personal access token (`TRIGGER_ACCESS_TOKEN`) covers all envs and projects: one token for all env-var pushes and deploys.

## Scars & gotchas

- **`deploy` fails at the depot builder with `unauthenticated: Invalid token`, use the `@latest` CLI, not a pinned older one.** A pinned CLI (seen at 4.4.4) can fail the remote depot build step with `unauthenticated: Invalid token` even though `trigger.dev login` reports "already logged in" and there is no self-hosted Docker in play. Fix: `npx trigger.dev@latest deploy`. The latest CLI refreshes the depot auth the pinned one leaves stale. (Note: this can conflict with the CI version-pin advice below; when both bite, match the CLI to the installed SDK first, then try `@latest` if the depot auth is the failure.)

- **Triggering a task from outside the SDK (e.g. a serverless function), REST, not the SDK.** `POST https://api.trigger.dev/api/v1/tasks/{taskId}/trigger`, header `Authorization: Bearer <TRIGGER_SECRET_KEY>` (the environment's prod secret key, `tr_prod_...`), body `{ "payload": { ... } }`. Returns the run handle. Do it best-effort from the edge (log and swallow a failed enqueue): the task itself owns retries/pacing, so the caller must never block the user on it.

- **Org membership is account-wide, not per-project** - A Trigger.dev org member can see *every* project in the org (runs, env vars, logs); there is no per-project member scoping below Enterprise. If one org holds multiple clients' or products' projects, inviting a developer to the org hands them all of that automation infra + secrets. Isolate sensitive work in separate orgs, or grant a scoped personal access token for CI/deploy instead of a full org invite.

- **Worker overwrite kills other codebases' scheduled tasks** - Deploying a second codebase's tasks into a project ref already used by another codebase makes the second worker the latest deployment, so the first codebase's tasks are no longer registered and its crons silently stop firing. The task simply isn't in the current worker. Fix: one project ref per codebase, no exceptions.

- **Missing prod env vars cause silent cron failure** - When required env vars are absent from Trigger.dev prod, a task can throw inside a helper, catch it non-fatally, return "ok", and write no alert. The downstream effect (e.g. an empty column, a missing link) may go unnoticed for weeks. Fix: push vars via the personal access token and add `syncEnvVars` so local `.env` deploys automatically.

- **Free plan $5/mo cap silently freezes all runs** - A large broadcast (700+ recipients) can exhaust the free credit. Every cron and webhook run then sits QUEUED for hours. The status page stays green, and `list_runs`, `whoami`, `get_current_worker` MCP tools expose no billing state. The only indicator is a dashboard banner. Fix: upgrade to a paid plan. Mitigation: a weekly billing-burn cron + alert before the cap is hit.

- **Non-fatal catch on high-value tasks is an invisible outage** - When a task catches all exceptions and returns a success-shaped object, `onFailure` never fires, no alert surfaces, and the task shows green forever while doing nothing. Pattern fix: throw on real/unexpected errors; soft-skip only known, expected transients; add an explicit guard (`if (registrants > 0 && triggered === 0) throw`) so a 100%-skip rate counts as a failure.

- **node-22 required for Supabase Realtime** - A deploy on the default Node 21 runtime fails immediately with "Node.js 21 detected without native WebSocket". `@supabase/supabase-js` Realtime needs native WebSocket, available from Node 22+. Fix: `runtime: "node-22"` in `trigger.config.ts`.

- **A large blast can starve a time-critical cron (queue design failure)** - A blast fan-out running on a shared queue with `concurrencyLimit: 1`, where each child takes several seconds, produces head-of-line blocking (e.g. 700 children x 3.5s = ~40 min). A time-critical reminder cron fired on time can have its children sit behind blast children for many minutes. Emergency fix: an early-return in blast children to drain the queue. Permanent fix: blasts get a dedicated low-priority queue, never shared with time-critical sends.

- **`syncEnvVars` without an allowlist pushes every local secret to prod** - Scope `syncEnvVars` to an explicit prefix list (e.g. `META_`, `LEADS_SHEET_ID`, `GOOGLE_CLIENT_EMAIL`, `GOOGLE_PRIVATE_KEY`). This prevents unrelated creds leaking into a Trigger.dev prod env that another codebase in the same org could potentially read.

- **Idempotency keys survive deploy cascades** - When several deploys happen in rapid succession, aggressive idempotency (30d TTL on keys like `import-15k-${sourceId}` and `blast-${flowId}-${contactId}`) means re-running any parent is a no-op for already-processed children: zero duplicate creates, zero double-sends across the cascade. Make this the default for any fan-out or import task.

- **Mirror columns return null for `.text` and `.value`** - Monday mirror columns return the real value only in `display_value` (inside `... on MirrorValue { display_value }`). A cron reading phone via `.text` can get null for all registrants, trigger 0 messages, and return success with no alert. Fix: include the `MirrorValue` fragment in every GraphQL query that touches a mirror column.

- **`deploy` over a weak uplink (mobile hotspot) dies at the depot build - `TimedOut: Error building image` / `keepalive ping failed to receive ACK`** - the deploy ships a full build context (~46MB for one small repo) to the remote depot builder; on a hotspot the transfer crawls (8+ min) and the builder connection times out. Hit twice in one day on two different repos. Mitigations: keep heavy files (brand PDFs, media) out of the repo/context via .gitignore (a 16MB PDF nearly doubled the context), and retry from a stable network - the same deploy succeeds unchanged. A TIMED_OUT deployment leaves the previous worker serving, so runs keep working on the old task code; check `GET /api/v1/deployments` for the real status because the CLI can also just hang (kill the local `depot build` process).

- **Pin the `npx trigger.dev` CLI version to match the SDK** - `npx trigger.dev@latest` picks the newest CLI regardless of the installed SDK version. If they diverge, the deploy aborts with "Version mismatch detected while running in CI". Re-validated live: `@latest` (4.5.9) against SDK 4.4.6 aborted; pinned 4.4.6 deployed clean, including the depot build. Pin the invocation: `npx trigger.dev@<version>` where the version matches the `@trigger.dev/sdk` entry in `package.json`. Order of operations vs the depot `Invalid token` scar above: **pin to the SDK version first; only switch to `@latest` if the pinned CLI hits the depot `unauthenticated: Invalid token` failure** (and then also bump the `@trigger.dev/*` packages to match).

- **GitHub Actions deploys: always pass `--skip-sync-env-vars`** - The CI runner has no `.env` file so `syncEnvVars` fails or pushes nothing. Add `--skip-sync-env-vars` to the deploy command in CI. Side effect: edge/env changes and new secrets do NOT reach Trigger.dev prod automatically. They require a local deploy or a manual edit in the Trigger dashboard to take effect.

- **`dirs:["./src/modules"]` auto-registers every task file under that tree** - Each module owns its tasks and private helpers; shared utilities live in `shared/` only. Never import from a sibling module. This is a clean monorepo boundary rule.

- **3MB per-run payload cap, base64-encode binary assets** - The run payload limit is 3MB. Binary uploads (signature PNGs, files) must be base64-encoded before inclusion. Payloads over 512KB auto-offload to object storage but still count toward the cap.

- **`client.defineJob()` (v2) silently breaks in the v4 SDK** - Any task using the old `client.defineJob()` + `io` argument pattern does nothing in v4. Always use direct imports from `@trigger.dev/sdk`: `task`, `schemaTask`, `schedules.task`. The SDK import path is the only correct entry point.

- **Minute-bucket idempotency key deduplicates double-clicks only** - `sha256(phone + url + minute-bucket)` catches the same person submitting twice within the same minute. It does NOT deduplicate the same lead re-submitting hours or days later. Longer-window dedup is a process concern, not the task's. Scope the dedup window to match the actual risk.

- **`TELEGRAM_ALERT_CHAT_ID` env var drives onFailure routing** - The `onFailure` hook in `trigger.config.ts` reads `TELEGRAM_ALERT_CHAT_ID` to route failure alerts. Set per-project to a dedicated alerts chat. Never point it at a shared conversational bot.

- **Trigger.dev is the safe home for broad-permission secrets** - Secrets like a Google Service Account JSON key or an LLM API key must NOT live on a user-facing edge (e.g. Netlify Functions). Trigger.dev tasks are server-side only, isolated per run, and add built-in retries plus a run-monitoring UI. Route any task needing broad permissions through Trigger.dev, not edge functions.

- **`syncEnvVars` + a secret store keeps secrets off the Trigger dashboard UI** - Pull secrets from a local secret store (e.g. macOS Keychain) at deploy time via `syncEnvVars` and inject them into the Trigger.dev project env. This avoids pasting values into the web UI and keeps the secret store as the single source of truth.

- **Run `metadata` updates can lose the final tick (race with run completion)** - A broadcast can finish with return value `sent: 55` while the dashboard metadata shows `sent: 54, progress: 54`. `metadata.set/increment` flushes are async and the last flush can race the run finishing, so dashboard metadata silently under-counts. Treat the task's return value as the authoritative tally; metadata is best-effort progress UI only, never reconcile or alert off it.

- **`trigger.dev deploy` ships the ENTIRE working tree, including uncommitted changes** - there is no clean/stash guard. Run `git status` before deploying and flag anything dirty you didn't write.

- **`schemaTask` SILENTLY STRIPS unknown payload keys, a safety filter that exists only in local code (not the deployed version) is dropped with no error.** A task triggered with `onlyPhones: ["<one number>"]` as a single-recipient preflight before a full-board blast: the task's Zod schema strips unrecognized keys (default `z.object` behavior, NOT `.strict()`), so `onlyPhones` is dropped, `payload.onlyPhones` is `undefined`, the allowlist filter never runs, and the "preflight" blasts most of the real list before it is caught. Root cause: `onlyPhones` was added to the local task file but the LIVE deployment predated it, so its schema had no such field. **The trigger call succeeds and the dashboard even echoes the key back in the payload, nothing errors.** Lessons: (1) verify the DEPLOYED schema before relying on a param, `get_task_schema` shows exactly what the live worker accepts. (2) For a single-recipient test, don't trust an allowlist param, use a mechanism you can confirm live, or redeploy first. (3) An `excludePhones` param that IS in the deployed schema works, so recovery = re-run excluding the already-sent numbers (dry-run to confirm the count first). (4) A silently-stripped safety filter is strictly more dangerous than a rejected one, prefer `.strict()` on any schema where an unknown key should abort rather than no-op.

- **A `tr_dev_` secret key used from the edge leaves every run QUEUED forever** - dev-environment runs only execute while a local `trigger dev` session is connected, and a *deployed* task runs in the PROD environment. So an edge caller (a serverless function, a webhook handler) that triggers with the DEVELOPMENT secret key (`tr_dev_...`) enqueues into an environment with no worker attached: the runs sit QUEUED indefinitely with no error at all. The page returns 200, the caller looks healthy, and the downstream alert / CRM move / draft never fires. `list runs` shows them QUEUED - not EXPIRED, not FAILED - which is the tell. Fix: the edge caller's `TRIGGER_SECRET_KEY` must be the **prod** key (`tr_prod_...`, Project -> API Keys -> Production), then redeploy the caller so it picks up the change. Distinct from the free-plan billing freeze above, which also parks runs QUEUED.

- **The global `onFailure` hook fires on `AbortTaskRunError` too** - `AbortTaskRunError` is the correct way to end a task on a permanent, non-retryable condition (record has no email, no CRM id, unknown id), but it still counts as a **failed run**, so an `onFailure` alert hook posts a "task failed" message for every expected skip. On normal traffic where a meaningful share of records legitimately abort, that floods the alerts channel and trains everyone to ignore it. Guard the hook to alert only on real failures: `if (error?.name === 'AbortTaskRunError') return;` at the top of `tasks.onFailure`.

## Conclusions / best practices

- One project ref = one codebase = one `trigger.config.ts`. This is the single most load-bearing rule. Violating it silently kills schedules.
- `syncEnvVars` with an allowlist on every project. Never deploy without it.
- `node-22` as the default runtime. No reason to use anything lower.
- Idempotency keys everywhere, 30d TTL for imports and fan-outs. Makes redeploys and retries safe by default.
- Blasts and bulk fan-outs get their own dedicated queue. Never share with time-sensitive reminder queues.
- Throw on unexpected errors in crons. Soft-skip only known transients. Add a "did it do anything?" guard if the task can silently no-op.
- Scope a Telegram alerts bot per project (`onFailure` hook in `trigger.config.ts`). Distinct from any shared conversational bot.
- Sheet-as-state: a Google Sheet deduped by a key column (e.g. `lead_id`) can replace a DB for idempotent state. Self-backfills on first run, safe to re-run on retry.
- Know when NOT to use the stack: a hard constraint that rules out any remote server (e.g. an on-prem-only requirement) can push you to a local scheduler (launchd + SQLite) instead of Trigger.dev.
- Deploy from a non-interactive/CI shell aborts with "Version mismatch detected while running in CI" when the CLI (`npx trigger.dev@latest`) differs from the installed `@trigger.dev/*` packages. Pin the CLI to the installed version, e.g. `npx trigger.dev@4.4.4 deploy`.
- `@supabase/supabase-js` pulls in `realtime-js`, which needs a native `WebSocket`; on the default runtime a task throws "Node.js detected but native WebSocket not found." Set `runtime: "node-22"` in `trigger.config.ts`.
