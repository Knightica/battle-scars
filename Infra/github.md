# GitHub

**Use for:** Source hosting, `gh` CLI automation (repo create/transfer, collaborators, API), deploy keys for Cloudways Git Deploy, and the GitHub App link that drives Netlify continuous deployment.
**Status:** Active

**TL;DR:** Repos belong under an **org**, not a personal account - `gh` is typically authed as a personal user, so `gh repo create <name>` lands under personal unless you transfer it. Transfers are **async** (`gh api repos/<owner>/<name>/transfer -f new_owner=<org>`) and you must **re-point every local clone's remote** afterward. For CD: Cloudways uses a read-only **deploy key**; Netlify needs the **GitHub App** linked (REST-API-created sites aren't, and CD breaks silently). Keep work repos **private**.

## Setup & access

- Confirm the authenticated `gh` identity with `gh auth status` / `gh api user -q .login`. The CLI is usually authed as a personal account even when the canonical home for repos is an org.
- Decide the canonical org for each repo up front. The personal account is just the operator identity, not the owner of record.
- Keep work repos **private**. Never create a public repo for client or internal work.
- Collaborators are added per-repo with least privilege: `gh api -X PUT repos/<owner>/<repo>/collaborators/<user> -f permission=push` (or `gh repo add-collaborator`). Prefer `push`, not `admin`.

## Scars & gotchas

- **A force-push does NOT purge the orphaned commits from GitHub - they stay fetchable by direct SHA until server-side GC.** Rewriting history (`git reset` + `push --force-with-lease`) removes a bad commit from every branch, but GitHub keeps the dangling commit reachable at `github.com/<owner>/<repo>/commit/<sha>` (and via API/fetch-by-SHA) for an indefinite period, plus it can persist in PR references and forks. So a history rewrite kills discoverability (nothing links to it), not existence. For a guaranteed purge of leaked content, ask GitHub Support to run GC on the repo - and rotate any leaked credential regardless; treat it as exposed.

- **On a PRIVATE repo, a "read" collaborator cannot open a pull request at all.** Read grants no push, so they cannot create a branch in the repo - and the usual escape hatch, forking, is closed too because "Allow forking of private repositories" is off by default at the org level. The result is a collaborator who can see everything and contribute nothing, with no error message explaining why. **"Triage" does not help either** - it adds issue/PR management, not `push`. The minimum permission for someone to open a PR from a repo branch is **`write`**. Set it with `PUT /repos/{owner}/{repo}/collaborators/{user}` + `permission=push`; for an existing collaborator it applies immediately with no invite to re-accept and no extra seat billed. Verify with `GET /repos/{owner}/{repo}/collaborators/{user}/permission`.

- **Repository ruleset `bypass_actors` DOES accept `actor_type: "User"` - you do not have to promote anyone to grant an exemption.** The REST docs and most examples show only `RepositoryRole`, `Team`, `Integration`, `OrganizationAdmin` and `DeployKey`, which pushes you toward the ugly workaround of bumping a person to `maintain`/`admin` just so a role-based bypass covers them. Passing a plain numeric user id works:

  ```json
  "bypass_actors": [{ "actor_id": <numeric-user-id>, "actor_type": "User", "bypass_mode": "always" }]
  ```

  Get the id from `GET /users/{login}` -> `.id`. This is the clean way to say "everyone must PR into this branch except these two humans" without touching anyone's permission level. Note rulesets on **private** repos need the **Team** plan or above.

- **`gh repo create` lands under the personal account, not the org** - because `gh` is authed as a personal user, `gh repo create <name>` (without an org prefix) creates the repo under that user. Repos created this way have to be transferred to the org afterward. Fix: either create directly under the org (`gh repo create <org>/<name>`) or transfer immediately after.

- **Repo transfer is async - the response still shows the old owner** - `gh api -X POST repos/<owner>/<repo>/transfer -f new_owner=<org>` returns a 202 with the repo body still listing the *old* owner. The move completes server-side a few seconds later. Don't treat the response as the final state; poll `gh repo view <org>/<repo>` until it resolves before re-pointing remotes.

- **After a transfer, every local clone's remote must be re-pointed** - the transfer moves the remote repo but local clones still point at the old `<old-owner>/<repo>.git` URL. Run `git remote set-url origin https://github.com/<org>/<repo>.git` in each clone. GitHub **auto-redirects** the old URL to the new owner, so a clone you forget keeps working - but don't rely on the redirect; re-point so the canonical URL is explicit. Sweep for stragglers: `find ~ -type d -name .git | while read g; do git -C "$(dirname "$g")" remote get-url origin; done | grep <old-owner>`.

- **A repo can have no local clone even when its code lives in your tree** - some "repos" are scripts tracked *inside* another repo (a subtree), not a separate checkout, so they have no remote to re-point after transfer. Confirm whether a "repo" is an independent clone or a subtree before hunting for its remote.

- **REST-API-created Netlify sites have no GitHub App installation_id - CD breaks** - a site created via the Netlify REST API is not linked to the GitHub App (no `installation_id`). First deploy attempts a git clone and fails with "Host key verification failed". Fix: re-link the repository manually in the Netlify UI so the GitHub App installation is attached. Both the Netlify MCP and REST API fail silently on this. (See also Infra/netlify.md.)

- **Cloudways Git Deploy needs a read-only deploy key + org-level enablement** - Cloudways blocks interactive SSH (ForceCommand), so deploy via the Cloudways Git Deployment feature: wire a **read-only GitHub deploy key** on the repo and set `deploy_keys_enabled_for_repositories` to true at the GitHub **org** level if it isn't already. Then trigger deploys via the Cloudways REST API. (See also Infra/cloudways.md.)

- **GitHub Actions deploys have no `.env` - pass `--skip-sync-env-vars`** - a CI runner has no local `.env`, so a Trigger.dev `syncEnvVars` step fails or pushes nothing. Add `--skip-sync-env-vars` to the deploy command in CI. Side effect: new secrets/env changes then do NOT reach Trigger.dev prod from CI - they need a local deploy or a manual dashboard edit. (See also Automation/trigger-dev.md.)

- **A pending org invite grants nothing, and the invitee does NOT appear in `orgs/<org>/members`** - someone who never accepted their invitation is invisible to the members list, but shows up in `orgs/<org>/invitations` and, if they were also added to a repo, in `orgs/<org>/outside_collaborators`. Before assuming a teammate has access, check all three endpoints. A developer can sit on an unaccepted invite from onboarding for months without anyone noticing, because the one list people actually look at doesn't mention them.

- **Repo collaborator adds apply INSTANTLY for org members but sit PENDING for outsiders** - `PUT repos/<owner>/<repo>/collaborators/<user>` returns an empty body and takes effect immediately when the user is already an org member, but returns an invitation object for anyone else. Same call, two different outcomes. Verify with `repos/<owner>/<repo>/collaborators` (active) **and** `repos/<owner>/<repo>/invitations` (pending) - a clean-looking collaborator list can still be missing someone.

- **`enforce_admins: true` deadlocks a two-approver CODEOWNERS setup** - GitHub does not let anyone approve their own pull request. With `enforce_admins: true`, `require_code_owner_reviews: true` and only two code owners, each owner's PR needs the *other* owner to be available. On a small team with part-time people that is a hard block, not an inconvenience. Set `enforce_admins: false` so owners retain a direct-push path, and use `restrictions.users` to keep everyone else off the branch.

- **`CODEOWNERS` must have NO file extension - `CODEOWNERS.md` is silently ignored** - GitHub only reads a file named exactly `CODEOWNERS`, in the repo root, `.github/`, or `docs/`. Renaming it to `CODEOWNERS.md` (an easy thing to do when tidying a docs-heavy repo, and it still renders nicely on GitHub) makes `require_code_owner_reviews` match zero owners - so the branch stays protected while the "an owner must approve" half quietly stops applying. Nothing errors and no warning appears. Verify with `gh api repos/<owner>/<repo>/contents/CODEOWNERS` (must resolve) plus `branches/<branch>/protection` -> `.required_pull_request_reviews.require_code_owner_reviews`. The documented `codeowners/errors` endpoint can 404 depending on token scope, so don't rely on it as the check.

- **A successful push to a protected branch does NOT mean protection is off** - when the pusher holds a bypass allowance, `git push` succeeds and the remote prints `remote: Bypassed rule violations for refs/heads/main:` followed by the rule it ignored (`Changes must be made through a pull request`). It is easy to read that as a clean push. Read the remote output, not just the exit code: if you see "Bypassed rule violations", protection is live and you just walked through it.

- **Branch protection with CODEOWNERS works on PRIVATE repos on the Team plan** - the old "protected branches on private repos need Enterprise" rule no longer applies. Confirm with `orgs/<org>` -> `.plan.name` (`team` is enough), then `PUT repos/<owner>/<repo>/branches/<branch>/protection`. Send `required_status_checks: null` and `restrictions: {users, teams, apps}` explicitly - the endpoint rejects a partial body.

## Conclusions / best practices

- Create repos directly under the org: `gh repo create <org>/<name> --private`. Don't create-then-forget under a personal account.
- After any transfer: poll until the new owner resolves, then re-point every local clone and sweep the home tree for stale remotes.
- Keep the `gh` auth as the operator's personal account but treat the org as the source of truth for ownership.
- All work repos private; add collaborators per-repo with least privilege (`push`, not `admin`).
- For CD: Cloudways = read-only deploy key + Git Deploy; Netlify = GitHub App link (verify `installation_id` exists, especially for API-created sites).
- In CI, never assume a `.env` exists - skip env-sync steps and push secrets through the platform's own mechanism.
- Access audits check three lists, not one: org members, org invitations, and outside collaborators. Same for a repo: collaborators AND invitations.
- After touching a `CODEOWNERS` file at all, verify three things: the file resolves at `contents/CODEOWNERS`, `require_code_owner_reviews` is still true, and every handle in it is an actual collaborator. An extension, a typo'd handle, or an owner without repo access all degrade to "no owners matched" without failing anything.
- Shared-asset repos: owners get `admin`, everyone else gets `push` plus a protected `main` (PR required, 1 approving review, `require_code_owner_reviews`, `restrictions.users` limited to the owners, force-push and deletion off). Leave `enforce_admins` false so the owners never deadlock each other.
