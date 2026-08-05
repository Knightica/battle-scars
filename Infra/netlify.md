# Netlify

**Use for:** Static and SSR site hosting, Netlify Functions for webhook endpoints, branch preview deploys, and DNS/domain management.
**Status:** Active

## Setup & access

- Use the Netlify MCP for most project and deploy work. Fall back to the raw Netlify REST API for the silent-failure cases documented below.
- Each site in a monorepo gets its own base directory.
- Each app folder gets its own `netlify.toml` with an `ignore` rule so a site only rebuilds when its own folder changes.
- Branch preview: a `dev` branch auto-deploys to `dev--{site-name}.netlify.app` for supervised WIP before promoting to `main`.
- Netlify Functions live in `netlify/functions/` alongside site code. Common uses: webhook callbacks and alert pings.

## Scars & gotchas

- **The Netlify MCP has no domain management; attach custom domains via the raw API** - The MCP exposes env vars, deploys, forms and project reads, but nothing for domains. `PATCH https://api.netlify.com/api/v1/sites/{site_id}` with `{"custom_domain": "<apex>", "domain_aliases": ["www.<apex>"]}` (Bearer token from a secret store) works first try, even before DNS points anywhere; TLS auto-provisions once the A/ANAME + CNAME records resolve. Verified live: attach, DNS records ~15 min later, HTTPS 200 within the half hour with zero further action.

- **Build settings (base dir, branch allow-list) are also API-only** - the MCP has no build-settings operations either. `PATCH https://api.netlify.com/api/v1/sites/{site_id}` with `{"build_settings": {"base": "<monorepo folder>", "allowed_branches": ["main", "dev"]}}` sets the base directory and enables branch deploys in one call. Useful when a project was created bare in the dashboard and needs wiring to its monorepo folder + `dev` branch deploys.

- **Publish dir `"."` serves every file in the repo folder, including internal `.md` docs - and a header claiming otherwise proves nothing.** When a site's publish directory is the app folder itself (a flat HTML site with no build step), Netlify serves *everything* in it as a static asset. Two internal docs carrying analytics ids, service-account emails, env-var names and the full event plan sat in the publish dir; a `[[redirects]]` bounce had been added for one of them only, and the other was left returning **HTTP 200** in production for a day - while its own first paragraph read "Internal doc. Not served publicly (a Netlify rule 404s it)." The doc's self-description was never verified against the deployed site. Two lessons: (1) the bounce is **per file**, so every internal doc added later needs its own rule, and a glob is safer - `from = "/*.md"` - than enumerating; (2) **verify with `curl -o /dev/null -w "%{http_code}" https://<domain>/<file>`**, never trust a comment in the file. Same trap applies to `.env.example`, `README.md`, and any `docs/` folder living under the publish dir.

- **Bulk env-var create (`POST /accounts/{account_id}/env?site_id=...`) 422s when a SECRET var uses `context: "all"`** - the API rejects the whole batch with `Secrets are not allowed to have 'All contexts' context. Explicitly set which contexts to use.` A var with `is_secret: true` must enumerate explicit contexts (one values entry per context: `production`, `deploy-preview`, `branch-deploy`, `dev`); non-secret vars can keep a single `{"context": "all"}` entry. Verified seeding a new site with two secret vars (per-context) + one plain var (all) in one POST, which returned 201. This raw-API path also keeps the values out of the transcript (read them from a secret store in the same shell), unlike `netlify env:set`/`env:import` which echo.

- **`ignore` rule cancels the go-live deploy after a squash-merge (cross-branch build cache)** - A per-app `ignore = "git diff --quiet $CACHED_COMMIT_REF $COMMIT_REF -- ."` skips a build when the app folder is unchanged since the LAST BUILT commit - and `$CACHED_COMMIT_REF` is the site's last build across ANY branch, not the current branch. Promoting `dev`->`main` by squash-merge produces a `main` commit whose app-folder tree is byte-identical to the last `dev` preview build, so Netlify cancels the production deploy with an errored deploy: "Canceled build due to no content change". The site keeps serving the old published deploy (looked like the merge did nothing). Fix: trigger a cache-cleared production build (`POST /api/v1/sites/{id}/builds {"clear_cache":true}`), or harden the ignore to always build the production branch: `ignore = "[ \"$BRANCH\" = main ] && exit 1 || git diff --quiet $CACHED_COMMIT_REF $COMMIT_REF -- ."` (exit 1 = build).

- **Secret env vars are deploy-bound, and pasted quotes/`KEY=` become part of the value.** Netlify *secret* env vars are baked at build time (unlike plain vars, which are runtime-injected), so changing one (e.g. `TRIGGER_SECRET_KEY`) does nothing at the function endpoint until the site **rebuilds**. Three traps hit in one session: (1) the value was scoped "different value per context" with the **Branch-deploys** slot empty, so the `dev` branch deploy read it as missing (`no_key`) - set the value for the context you actually run in. (2) Pasting the value **with** surrounding quotes or a `KEY=` prefix stores them literally, so the function sent `Bearer "tr_prod_..."` and got `http_401 Invalid API key` - paste the bare value only. (3) The site's `ignore` rule (scoped to a subdir) means an **empty/no-op commit won't rebuild**, so the new secret never binds - bump a marker comment in a file under that subdir to force the rebuild.

- **`netlify env:set KEY VALUE` echoes the value to stdout - a secret leak** - the CLI prints `Set environment variable KEY=<full value> in the production branch` on success, so setting a secret puts it in the terminal/log/transcript. When sourcing the value from a secret store to avoid a paste-leak, still redirect: `netlify env:set KEY "$VAL" >/dev/null 2>&1 && echo ok`. If it already printed, rotate the secret. Verify presence without exposing values via `env:list --json | python -c "'KEY' in json"` (keys only).

- **`--context production` may not reach the runtime; set env for ALL contexts** - a `netlify env:set KEY VAL --context production` reported success but the key did NOT appear in `env:list --json` and wasn't readable at runtime. Setting it with **no** `--context` flag (all contexts) made it show up and take effect. Prefer all-contexts for anything the deployed site reads, unless you specifically need per-context values. (Still requires a redeploy to go live - see below.)

- **Env var changes need a redeploy to reach Functions** - Functions read env vars from their deploy context, so changing a var (via REST API, MCP, or the UI) does NOT take effect on already-deployed functions. Symptom: updated env still returns the old value at the function endpoint until a rebuild. Fix: after any env change, trigger a fresh build (`POST /sites/{id}/builds`) and wait for `state: ready` before testing.

- **MCP silent failure: secrets with "all" context are dropped** - Setting env secrets via the Netlify MCP with context "all" silently does nothing. No error is returned; the secret is simply never written. Set each secret per-context (production, deploy-preview, branch-deploy) via the raw Netlify REST API.

- **MCP silent failure: domain aliases are a no-op** - The Netlify MCP has no domain-alias operation. Calling any alias-adjacent tool to add subdomains (e.g., `join.example.com`) does nothing. Use `POST /api/v1/sites/{site_id}/aliases` directly via the Netlify REST API with a securely-stored token.

- **MCP silent failure: SSL provisioning is a no-op** - SSL certificate re-issuance via MCP does not trigger. Use the raw REST API (`POST /api/v1/sites/{site_id}/ssl`) after adding domain aliases.

- **Domain aliases, not netlify.toml rewrites, handle subdomains** - netlify.toml `[[redirects]]` routes paths, not hostnames. To route `join.example.com` to a specific app, add it as a Netlify domain alias on the target site.

- **Build OOM on complex TypeScript projects** - Large type graphs (e.g., `googleapis` typings pulled in by a `trigger/` folder) will OOM the Netlify build sandbox and return exit code 2 with no clear error. Fix: set `NODE_VERSION=22` (or recent LTS) in Netlify env, and exclude the heavy folder from `tsconfig.json` using `exclude`. Builds can silently fail for an extended period before root cause is identified.

- **Split Test silently re-routed 100% of production traffic to a branch deploy** - A Netlify Split Test left configured to send all production traffic to a `dev` branch deploy served the wrong content with no error or alert. Fix: unpublish the split test via `POST /traffic_splits/{id}/unpublish` and purge the edge cache. Rule: never set up production traffic experiments without an explicit decision and active monitoring.

- **Netlify Blobs auto-config can fail in Functions v1** - The error "environment has not been configured to use Netlify Blobs" fires when a v1 Function handler calls the Blobs API without explicit context. Blob writes 500 on the browser-polling path. Fix: migrate the handler to Functions v2 syntax (which auto-injects blob context) or pass `siteID` + `token` explicitly in the Blobs constructor.

- **`_redirects` takes precedence over netlify.toml for static rewrites** - When a `_redirects` file is present, its rules are evaluated before `netlify.toml` redirects. Use this to hard-archive a site or funnel: a `_redirects` force-301 (including payment endpoints) redirects all traffic to home without touching the main toml config.

- **REST-API-created sites have no GitHub App installation_id - CD breaks** - Sites created via the Netlify REST API are not linked to the GitHub App (no `installation_id`). On first deploy Netlify attempts a git clone and hits "Host key verification failed". Fix: re-link the repository manually in the Netlify UI for continuous deployment to work. The MCP and REST API both fail silently on this.

- **Netlify Forms only work on the deployed site, not local serve** - Form detection and submission do not work during `netlify serve` or `netlify dev`. Keep the hidden `form-name` input, honeypot field, and `data-netlify="true"` attribute on every form. Any field name or attribute change re-triggers form detection on the next deploy.

- **Raw-body HMAC pattern eliminates canonical-form mismatch bugs** - The browser HMAC-signs the exact canonical JSON bytes it is about to POST; the receiving Netlify Function HMACs those same raw received bytes without re-canonicalizing server-side. This eliminates the class of bugs where the client and server stringify the same payload object differently. Give each form its own HMAC secret.

- **SSL cert blocks adding a new domain alias when a stale cert already covers a different alias** - Netlify SSL provisioning refuses to add a new alias when the existing cert was issued for a different single alias. Fix: toggle the problem domain alias off in Netlify settings, then back on, then force re-provision via `POST /api/v1/sites/{site_id}/ssl`. This issues a fresh multi-alias cert covering both names.

- **Netlify Blob as a fail-closed first layer in a duplicate guard** - Use a Netlify Blob as the fast first layer in a dual-layer duplicate guard (Blob check + a slower external-API pre-check). The Blob gives a near-instant deny before the slower call; both layers must return clean before proceeding. Fail-closed when either layer returns an error.

- **`env:get` returns EMPTY for secret env vars** - `netlify env:get KEY --plain` returns nothing when KEY is stored as a *secret* (write-only) var, so you can't read it back to (e.g.) curl-test an endpoint that uses it as an auth token. Test via the real caller (webhook) instead, or via the function's own logs.

- **`env:import` ECHOES the imported values in its confirmation table** - `netlify env:import <file>` prints a Key|Value table to stdout, so importing a secret dumps it into your terminal/transcript (a leak). Import secrets knowing this, and rotate if the transcript is shared. To set secrets, prefer the raw API or accept+rotate.

- **Function logs: `logs:function` is deprecated** - use `netlify logs --source functions --function <name> --since 10m` for **historical** invocation logs (last N minutes), which is what you want to debug a webhook after the fact. `logs:function` only streamed live. Note: the log shows the Lambda `Duration` line always, but `console.log/error` only appear if the code actually logged - a handler that swallows errors shows nothing, so add explicit `console.error` in catch blocks.

- **CLI auth from a secret store, non-interactively** - `NETLIFY_AUTH_TOKEN=$(security find-generic-password -s <keychain-entry> -w) npx netlify <cmd>` runs the CLI without an interactive login (the token lives in the OS keychain). Poll deploy state via `netlify api listSiteDeploys --data '{"site_id":"…","per_page":1}'` and grep for `ready <commit>`.

- **GitHub->Netlify webhook can silently miss a push** - the push succeeds but no deploy is created. Check the site's deploy list after pushing (state + title), not just git output. An empty commit (`git commit --allow-empty`) re-fires the webhook.

- **Env vars can hold DIFFERENT values per deploy context** - a `WEBHOOK_HMAC_SECRET` that intentionally differs on deploy-preview vs production means browser HMAC-signed form submits 401 silently on previews (fire-and-forget hides it). When a preview behaves differently from prod, dump env var contexts via the MCP `manage-env-vars getAllEnvVars` before debugging code.

- **MCP `manage-env-vars upsertEnvVar` returns HTTP 422 on an EXISTING key** - the MCP's "upsert" is a POST/create under the hood, so Netlify rejects it as a duplicate (`Failed to fetch API: 422`) when the key already exists. It only works for NEW keys. To CHANGE an existing var's value: `deleteEnvVar: true` for that key, then `upsertEnvVar` to recreate it at the new value. Then redeploy for functions to pick it up (see the redeploy scars above).

- **A team slug rename silently breaks every stored `teamSlug`, and `get-projects` fails with a bare 404** - Netlify team slugs are mutable. Renaming a team from its auto-generated slug (e.g. `yourname-a1b2c3d`) to something readable makes every team-scoped call using the old slug return `Failed to fetch API: 404`, with no hint that the slug is the cause. Anything holding a hardcoded slug - config files, monitoring, scripts - then scans an empty set and **reports clean**. That is the dangerous part: the failure mode is a false all-clear, not an error. Fix: resolve slugs at runtime via the team-services `get-teams` operation (returns `slug` plus a stable `id`), or store the immutable team `id` instead of the slug. A 404 from a team-scoped call means "team not found", not "no projects".

- **A 200 rewrite to a Function does NOT pass the rewrite's query string to the function** - a `netlify.toml` `[[redirects]]` with `from = "/s/:id"`, `to = "/.netlify/functions/landing?id=:id"`, `status = 200` does fire the function, but the function receives the ORIGINAL request context, so `event.queryStringParameters.id` is EMPTY - the injected `?id=:id` is dropped. Symptom: every valid short link 404s because the handler reads the id from the query. Fix: parse the id from `event.path` (`/s/<id>`), with the query string as a fallback. The `:id` placeholder only substitutes reliably for 30x redirects, not for 200-rewrites-to-functions.

- **Wrong build branch or unset base directory deploys nothing while still showing "ready", and a manual `--prod` deploy is reverted by the next git build** - a repo-connected site set to build `main` while the app code lives on a feature branch (base directory unset, so the app's own `netlify.toml` with its redirects and functions dir is never read) publishes an empty/404 site that still reports `state: ready`. Fix: set the **base directory** to the app subdir and point the **production branch** at the branch that actually has the code, or merge it down. Separately: a stop-gap `netlify deploy --prod` publishes immediately but is overwritten the next time a git push triggers a build from the connected branch, so always commit the change to the deploy branch too, or the manual deploy silently reverts.

## Conclusions / best practices

- For the MCP silent-failure cases (secrets, domain aliases, SSL) - always use the raw Netlify REST API.
- Set `NODE_VERSION` in Netlify env on any Next.js or TypeScript-heavy project. The default Node version on the build sandbox can be too old or too constrained for large type graphs.
- Exclude non-deployable folders (e.g., `trigger/`) from `tsconfig.json` to prevent build OOM.
- Use branch preview (`dev--{site}.netlify.app`) for all WIP. Promote to `main` only on explicit approval.
- No production traffic experiments (Split Tests, branch routing overrides) without a clear owner and monitoring in place.
- Netlify Functions are the right home for webhook endpoints that need to live alongside the site.
- **Env-var changes don't reach running functions until a new deploy** - Updating a site env var via the API (PUT /accounts/{slug}/env/{key}) changes nothing for already-deployed functions; values are baked at build time. Trigger a rebuild via `POST /api/v1/sites/{site_id}/builds` (empty JSON body) and wait for the deploy to reach `ready` (~25s for a static+functions site).
