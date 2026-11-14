# OpenPocket: plan

Status: **plan only, implementation-ready.** This is the plan for the public repo `kupferco/openpocket` (MIT). Nothing in the app, workflows, infrastructure, secrets or repo settings was changed to write it.
Legend: **Decided** = Dani chose it. **Recommended** = my proposal, needs a yes. **Verify** = a third-party fact I could not fully confirm; check it before building on it. Open questions are collected in section 17.

This version (13 Nov 2026, round 4, imported into this repo on 14 Nov 2026) records Dani's answers to the former open questions as decisions (section 17.1) and updates every section they affect. Section 17.2 holds only what is still open. "Viber" in this document is Dani's private predecessor repo (`kupferco/viber`), kept for the migration history (sections 11, 15, 16); step lists written as "Viber tasks" record how the work is run there. `docs/saas-plan.md` is merged into this plan and superseded (decision 18).

## 0. Summary

**OpenPocket** is the open-source product: *"Lovable without a keyboard"*: build and ship apps from your phone by voice. It lives in a **new public repo with clean history**, MIT. The public repo is `kupferco/openpocket` (created, MIT licence and README in place). `kupferco/viber` stays **private** as Dani's own instance (the predecessor, called "Viber" below).

Two ways to use it:

| | Standard user (hosted) | Committed user (self-host) |
|---|---|---|
| Runs on | The public hosted OpenPocket (launches after the open-source publish) | Their own OpenPocket (fork) |
| Setup | Add connector in Claude, sign in with GitHub, install the GitHub App | Fork, deploy to Vercel, paste own tokens |
| Coding engine | **Claude API key** (the user's own, runs in their own Actions) or **GitHub Copilot coding agent** on their Copilot plan | Own `claude setup-token` (personal use on their own instance only), Claude API key, or Copilot |
| App hosting | User's own Vercel | User's own Vercel |
| Database | User's own Neon | User's own Neon |
| Cost to user | Free to start (plus their Copilot plan, if not already paid) | Their own accounts |

Hard rules: OpenPocket **never** routes a third party's Claude Free/Pro/Max credentials ([Anthropic legal page](https://code.claude.com/docs/en/legal-and-compliance)); the hosted instance never accepts subscription tokens, and the README says so plainly (13). Dani hosts none of the users' apps and holds none of their data beyond OpenPocket's own account/task tables.

### Decisions (all Decided unless marked)

| # | Decision |
|---|---|
| 1 | Name **OpenPocket** (one word, capital O and P; it was the working name "Pocket"; conflicts in section 1). MIT licence. |
| 2 | New public repo, clean history. `kupferco/viber` stays private (Dani's instance, his Claude subscription, his projects). |
| 3 | One public repo is the single source of truth; only Dani pushes/merges, others fork and PR. |
| 4 | A self-hosted OpenPocket never opens PRs upstream by itself (section 12 guard). |
| 5 | Standard users: Claude custom connector with OAuth, "Sign in with GitHub" only, GitHub App installed on chosen repos, no personal tokens. |
| 6 | Standard users' engine: GitHub Copilot coding agent. Never Claude subscription tokens. Own Anthropic API key allowed as an alternative. |
| 7 | Hosting and database are the user's own Vercel and Neon, connected just in time (GitHub at sign-up, Vercel at first publish, Neon at first database need). |
| 8 | Opinionated stack: Node, Vite front end, Node functions on Vercel. Database only when a task needs it. Neon branch per preview. |
| 9 | Committed users self-host with a step-by-step guide; "Deploy to Vercel" button considered. |
| 10 | Article work out of scope. |
| 11 | Environments are **fixed**: always preview, staging, production (2.1). No custom-environments variant. |
| 12 | Two separate deployments of the same code: Dani's personal OpenPocket and the public hosted OpenPocket, each with its own staging and production (14.1). |
| 13 | Order: **publish open source first, hosted later**. |

All 27 answers from Dani (13 Nov 2026) are in section 17.1.
---

## 1. Name, repo and package availability

Checked on 12 Nov 2026 (anonymous GitHub/npm API calls plus web search):

| Item | Result | Verdict |
|---|---|---|
| GitHub `kupferco/openpocket` | Created (public, MIT) | Done |
| GitHub org/user `pocket` | exists (`users/pocket` → 200) | Taken, which is why the working name "Pocket" was extended; `kupferco/openpocket` is used |
| GitHub org `openpocket-app` | Not checked under the new name (`pocket-app` was free on 12 Nov 2026) | Possible org name |
| npm `pocket` (the old working name) | **taken** ("file system-based data store", v0.1.0) | Cannot publish under this name |
| npm `openpocket` / `@openpocket/*` scope | Not checked under the new name (`@pocket-app/cli` → 404 on 12 Nov 2026), scope not confirmed free (scopes are claimed by creating an org) | Reserve the scope early if an npm package is wanted |
| Mozilla **Pocket** (read-it-later) | Shut down 8 Jul 2025, API closed 8 Oct 2025. Mozilla still owns the mark. | Conflict risk: well-known name in the same "Pocket" word; the one-word "OpenPocket" is distinct but app-store/trademark search is still needed |
| **PocketBase** (open-source backend) | Active, developer audience, very close name | Search confusion risk |
| **Pocket Network** (POKT, crypto), **Pocket Casts** | Active products | Lower, but SEO noise; "OpenPocket" as one word helps |

**Decided: the product name is OpenPocket**, one word, capital O and P (it started as the working name "Pocket", which is no longer used). Only the **domain** is still open (e.g. `openpocket.<tld>`, check availability when buying). **No npm package in v1** (the app is deployed, not installed; a CLI later would be scoped `@openpocket/cli`). A **trademark check** (USPTO/EUIPO, software classes 9/42) is a launch blocker before the public repo goes public (step 0.1); if it fails, the fallback names are "OpenPocket Studio" / "OpenPocketship" (the plain "Pocket" is not a fallback). The old "Viber" name collision (messaging app) disappears with the rename; "Viber" remains only as the name of Dani's private predecessor repo.

**Repo owner: decided, `kupferco/openpocket`** (no new org). Consequences: the GitHub App is owned by the `kupferco` user account (transferable to an org later), CODEOWNERS is `* @kupferco`, and the old `kupferco` repos (including `kupferco/viber` after retirement) are tidied (made private or archived) **closer to launch**, not now (step D1/B6). Owner is a user, so "only Dani can push" is the default for a personal repo; no org-level rulesets.

---

## 2. Architecture in one picture

```
Claude app (phone) ──MCP over HTTPS + OAuth──▶  OpenPocket server  ◀── GitHub App webhooks
                                                  │   (MCP, web UI, hooks, sweep)
                                                  ▼
                                          OpenPocket DB (Neon)
                                                  │
        ┌─────────────────────────────────────────┼───────────────────────────────┐
        ▼                                         ▼                               ▼
 GitHub (user's repos)                    Vercel (user's account)          Neon (user's account)
 Copilot coding agent / Actions           preview per PR, prod on main     project + branch per preview
```

- **OpenPocket server** = what Viber is today (API + web UI + MCP + hooks), made multi-user. Same code runs hosted (Dani) or self-hosted.
- **Engines** are pluggable (`engine` column per task): `copilot`, `claude_actions` (Actions + API key, or subscription token on self-hosted only). The Claude API key engine is available **at launch**, runs in the user's own Actions like Viber today and needs no approval tap.
- **Hosting/DB** belong to the user; OpenPocket holds only revocable tokens to them.
- **Two separate deployments of the same code** (decision 27): Dani's **personal OpenPocket** (`OPENPOCKET_MODE=single`, self-hosted, his Claude subscription token, his projects) and the **public hosted OpenPocket** (multi-user; Dani uses only a test account there). Each has its own staging and its own production, its own database and its own secrets. Both run on **Vercel + Neon**; Viber stays on Cloud Run until retired.
- OpenPocket **creates new repos** for users from the starter template (needs the extra GitHub permission in 4.1).

### 2.1 Three environments, everywhere (Decided)

The same three environments Viber has, for OpenPocket itself and for every app OpenPocket builds:

| Environment | What it is | Deploys from | Who can trigger |
|---|---|---|---|
| **Preview** | One per pull request / task, own database branch, test data only | the task branch (PR) | automatic |
| **Staging** | Long-lived, for testing before release, own domain (`staging` subdomain) | `main` (every merge) | automatic on merge |
| **Production** | The real thing | the `production` branch, a build already proven on staging | **manual promote only**, gated by the spoken 4-digit code; never one voice command away |

**The set is fixed, not an option (Decided):** every app, on every plan, gets exactly these three, built with option A in 5.6. There is no custom-environments variant and no way to configure fewer or more; changing it means changing the source.
Details: OpenPocket itself in 14.1, apps OpenPocket builds in 5.6, journeys in 3.1 (steps 14a-14c) and 3.2 (environments paragraph), checklist in section 16.

---

## 3. User journeys

### 3.1 Standard user (hosted), screen by screen

Goal: first preview in under 10 minutes, five or fewer taps of setup before the first task.

| # | Screen / action | What the user sees | What OpenPocket does |
|---|---|---|---|
| 1 | **Landing page** `openpocket.<domain>` | One sentence, "Add to Claude" button with the connector URL and a copy button, 3-step illustration. | none |
| 2 | **Claude app** → Settings → Connectors → Add custom connector → paste URL | Claude shows the connector and a "Connect" button. | Serves MCP metadata (protected-resource + authorization-server metadata), supports dynamic client registration. |
| 3 | **OpenPocket sign-in page** (opened by Claude) | One button: **Sign in with GitHub**. Text: "We use GitHub only to find your repos and open pull requests." | OAuth authorize request from Claude is parked; redirect to GitHub App user authorization. |
| 4 | **GitHub authorization** | GitHub's own consent: OpenPocket (GitHub App) wants to verify your identity. | Callback creates the `users` row (+ `identities` for GitHub) and stores the encrypted user-to-server token and refresh token. |
| 5 | **GitHub App install** (straight after) | GitHub's "Install OpenPocket" page: choose "Only select repositories". | Setup URL returns to OpenPocket with `installation_id`; OpenPocket stores the installation and lists repos. |
| 6 | **OpenPocket: choose a project** | List of the repos they selected (or a prompt "none selected, pick again"). Tap one → "Use this repo". Option "Create a new repo for me" (new repo from the Vite starter template, created by OpenPocket under the user's account, needs the extra GitHub permission in 4.1). | Creates the `projects` row; writes `.openpocket/` marker, `CLAUDE.md`/`AGENTS.md`/`.github/copilot-instructions.md` rules via PR (not a direct push) and the optional workflow. |
| 7 | **Engine choice** | Two cards: "**Claude** (paste your Anthropic API key; runs in your own GitHub Actions, no tap needed)" and "**GitHub Copilot** (uses your Copilot plan; we found Copilot: ✔ / not enabled, with a link; Copilot Pro recommended, the free plan is limited)". | Copilot: checks that the `copilot-swe-agent` actor is assignable (GraphQL `suggestedActors`). Claude key: stored **only as a repo secret** `ANTHROPIC_API_KEY` through the GitHub App's write-only Secrets permission, never in OpenPocket's database. |
| 8 | **Back to Claude** (connector connected) | "OpenPocket connected." | Issues OpenPocket's own access + refresh token to Claude (OAuth). |
| 9 | **First task by voice**: "Make a to-do app" | Claude replies "Started task #1: …". | `start_task` → Copilot issue (section 6). |
| 10 | **Just-in-time: Connect Vercel** (first time there is a preview/publish) | Claude says "To show you a preview I need your Vercel account. Open this link." The link opens OpenPocket's "Connect Vercel" screen: one button, then Vercel's own consent. If they have no account, Vercel's sign-up is offered (sign-up with GitHub: one tap). | Vercel integration OAuth; creates a Vercel project linked to the repo (production branch `production`, staging on `main`, see 5.6), sets env vars. Stores token encrypted. **Plain-words notice at connect time:** "Vercel's free Hobby plan is for non-commercial use. If this app earns money you need Vercel Pro." Shown, not enforced. |
| 11 | **Preview ready** | Notification (ntfy/push, or Claude on next message) with the preview link; preview page auto-refreshes when a new build lands (section 5). | Reads the Vercel deployment for the PR head SHA; posts preview URL to the task. |
| 12 | **Needs a database** (only when a task requires one) | Claude: "This needs to save data. I recommend a free Neon database. Say yes to set it up." The user says yes → link to "Connect Neon" screen → Neon consent (sign-up if needed). | Creates a Neon project (user's account), a branch per preview (section 5), sets `DATABASE_URL`. |
| 13 | **Iterate** | "Make the button blue" → `continue_task`. | Same PR gets the follow-up (comment to Copilot). |
| 14a | **Merge to staging**: "Put it on staging" (or "Looks good") | Claude: "Merged. Staging is updating, link follows." Staging link on the task. | Merges the PR into `main`; Vercel builds the staging deployment (5.6); staging Neon branch gets approved migrations; OpenPocket reports "On staging" when READY. |
| 14b | **Test on staging** | The user tries the app at the stable staging URL. | Nothing; staging is not production and holds test data only. |
| 14c | **Promote to production**: "Deploy to prod" | Claude says what ships (checks, staging link, whether staging was checked) and reads a four-digit code; the user says it back. **Enhanced security is a toggle:** on hosted, the code goes by **WhatsApp** (Twilio number, approved message template, small per-message cost) instead of being read out. | First call never promotes. Second call with the code (bound to task, commit, hour) promotes the staging-proven build to production (5.6), then reports Live when the production deployment is READY. Migrations for production need their own spoken approval. |
| 14d | **Custom domain** (optional, Project settings) | A "Custom domain" field. After saving, Claude/OpenPocket says in plain words: "Add this record at your domain provider: type CNAME, name `www`, value … (or A record for the bare domain, value …). For staging, optionally add `staging.<domain>` the same way. I'll tell you when it works." | Adds the domain to the Vercel **production** project (and the staging subdomain to the staging branch domain, if asked), reads Vercel's required DNS records and verification state, rechecks until verified. OpenPocket never touches the user's DNS. |
| 15 | **Account page** (web) | Connected accounts (GitHub, Vercel, Neon) with **Disconnect**, engine and API key status (set/removed on the repo, never shown), **Delete my account**. When Copilot's quota runs out, a task shows plain words: "Your Copilot plan has no requests left for now. Upgrade to Copilot Pro, or add a Claude API key to keep going. | Section 9. |

Fewest clicks: steps 2-8 are a Claude connect (1), a GitHub consent (2), repo selection on GitHub (3), and returning. Vercel/Neon add one consent each, only when needed.

### 3.2 Committed user (self-host), screen by screen

| # | Step | Detail |
|---|---|---|
| 1 | Fork `kupferco/openpocket` (GitHub UI) | One click. |
| 2 | **Deploy to Vercel** button in the README (clone URL = their fork, project name prompt) | Env var prompts via `env=` list: `ENCRYPTION_KEY` (with a generator link), `ADMIN_PASSWORD`, `GITHUB_TOKEN`, `CLAUDE_OAUTH_TOKEN` or `ANTHROPIC_API_KEY`, `NTFY_TOPIC`. Verify the button supports Marketplace Neon storage in the same flow (Vercel's "integration" step lets the template add Neon, which injects `DATABASE_URL`). |
| 3 | Create a **GitHub fine-grained token** | Setup page in the deployed app shows a pre-filled token-creation link and the exact permission list. |
| 4 | **Claude**: run `claude setup-token` on a computer once and paste, **or** paste an Anthropic API key (or choose Copilot) | Subscription tokens are for **personal use on your own instance only**; the README says sharing one or running a service on it is not allowed. |
| 5 | Open the deployed URL, **sign in** with `ADMIN_PASSWORD` | `/setup` screen runs `doctor`: token scopes, DB reachable, engine present, ntfy test. |
| 6 | **Add connector** in Claude (URL shown on Setup) | Single-tenant mode uses the path-secret MCP URL (as today) or OAuth with the admin password; keep path secret in v1. |
| 7 | **Add a repo**: Projects → Add project | Writes workflow, CLAUDE.md block, secrets (the committed user's own PAT, so no GitHub App needed; the App is optional for self-hosters who register their own). |
| 8 | **Connect Vercel / Neon** | Paste tokens, or run the same OAuth flows if they register their own Vercel integration/Neon OAuth app (documented but optional; tokens are the default). |
| 9 | First task | As 3.1 step 9 onwards. |

Environments for a self-hoster (fixed, same as everyone): the OpenPocket **preview** is a Vercel preview per PR with its own Neon branch (their fork's PRs), **staging** is their fork's `main` on a staging domain with a long-lived Neon `staging` branch, **production** is the `production` branch promoted by hand with the spoken code. The setup page's `doctor` creates and checks all three; none can be skipped. The enhanced-security code goes by **ntfy** by default, or by the self-hoster's own Twilio (WhatsApp) if configured.

Target: under 30 minutes, five manual actions (fork, deploy button + env prompts, GitHub token, Claude token, connector).

---

## 4. Provider registrations and permissions

All of these are created **once by Dani** for the hosted OpenPocket. Self-hosters use PATs/tokens instead (or register their own).

### 4.1 GitHub App "OpenPocket" (hosted)

| Setting | Value | Why |
|---|---|---|
| Request user authorization (OAuth) during installation | **On** | Sign in with GitHub in one flow, and get the **user-to-server token** that the Copilot assignment API needs (installation tokens are not supported for assigning Copilot, **Verify** against the current [cloud agent API docs](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/cloud-agent/use-cloud-agent-via-the-api)). |
| Expire user tokens | **On** (8 h access, 6 mo refresh) | Revocation-friendly. OpenPocket refreshes and stores the rotating refresh token encrypted. |
| Callback URL | `https://<openpocket>/auth/github/callback` | Sign-in. |
| Setup URL + "Redirect on update" | `https://<openpocket>/github/installed` | Return after install. |
| Webhook URL + secret | `https://<openpocket>/hooks/github` | Receive PR, issue, check, installation events (replaces polling for hosted scale). |
| Repository: **Contents** R/W | | Read code, create branches, merge PRs, write the rules files. |
| Repository: **Pull requests** R/W | | Open, update, merge the PR. |
| Repository: **Issues** R/W | | Create the issue assigned to Copilot, read comments. |
| Repository: **Actions** R | (R/W only for engine `claude_actions`) | Read check runs, logs for failure reasons. |
| Repository: **Checks** / **Commit statuses** R | | Show check state on the task. |
| Repository: **Metadata** R | (mandatory) | |
| Repository: **Workflows** R/W | only with engine `claude_actions` | Writing `.github/workflows/openpocket.yml`. Skipped for Copilot users; fewer permissions at install. |
| Repository: **Secrets** (write-only) | only with the Claude API key engine | Store the user's API key as a repo secret (`ANTHROPIC_API_KEY`). OpenPocket can write it but never read it back; it is **not stored in OpenPocket's database** (decision 23). |
| Repository: **Administration** R/W | requested, needed for two things | (1) **Create a repo for the user** from the starter template (decision 15; **Verify** whether Administration, or the account-level "Administration"/"Repository creation" permission, is the one that works for a personal account: a GitHub App creates repos for a user via the user-to-server token and `POST /user/repos` or `POST /repos/{template}/generate`, which needs the App's **"Administration: write"** on the account; check this on a personal account in the spike A10b). (2) **Branch protection** on the production branch (4.6). OpenPocket does not change any other repo setting. |
| Account: **Email addresses** R | optional | Contact for security notices; can be skipped (use the GitHub noreply id). |
| Events | `installation`, `installation_repositories`, `pull_request`, `issue_comment`, `check_suite`, `workflow_run`, `github_app_authorization` | State changes and **revocation** (`github_app_authorization` fires when a user revokes). |

### 4.1a Quota and partner approvals

Copilot plan docs wording (decision 9): "Works with your Copilot plan; the free plan is limited, Copilot Pro recommended." When the quota is hit, OpenPocket explains it in plain words and suggests upgrading to Copilot Pro or adding a Claude API key.

**Verify**: how the Copilot cloud agent treats a GitHub App as issue creator and whether a PR opened by the agent needs a manual "Approve and run workflows" click (it does for Copilot PRs by default in many setups); this is the main UX risk (section 6).

### 4.2 Vercel integration

| Item | Detail |
|---|---|
| Kind | Vercel Integration (OAuth redirect flow) owned by Dani's Vercel team. **Verify** whether "Connect Vercel" is best as a Marketplace/Integration (requires Vercel approval for public listing) or "Sign in with Vercel" OAuth app + API calls. |
| Scopes | Create/read projects and deployments, read/write project env vars, read team/user. Not billing. **Domains**: add a domain to a project and read its verification/DNS records (for the custom domain field, 3.1 step 14d); OpenPocket never manages the user's DNS. |
| Why | Create the project linked to the repo (Vercel's Git integration then builds every PR as a preview and `main` as production, so OpenPocket needs no deploy workflow), set `DATABASE_URL`, read deployment URL and state. |
| Plan notes | Vercel Hobby is for non-commercial personal use; users shipping commercial apps need Pro (their cost). **Decided: tell users openly when they connect Vercel; do not enforce.** |
| Approval | **Apply now** for the Vercel integration listing; the first beta uses **user-pasted tokens** until the approval lands (decision 24). |

### 4.3 Neon OAuth app

| Item | Detail |
|---|---|
| Kind | Neon OAuth application (partner access). **Verify** availability/approval process on neon.com/docs/guides/oauth-integration. |
| Scopes | `urn:neoncloud:projects:create`, `projects:read`, `projects:modify` (branches), `orgs:read` (smallest set that works). |
| Why | Create a project in the user's own account, create/delete branches (preview, `preview-seed`, `staging`), read connection strings. |
| Approval | **Apply now** for the Neon OAuth/partner app; first beta uses user-pasted API keys until approved (decision 24). |

### 4.4 OpenPocket as an MCP OAuth authorization server

OpenPocket implements the MCP authorization flow itself (OAuth 2.1): protected-resource metadata, authorization-server metadata, dynamic client registration, PKCE, refresh tokens. Its `/authorize` page is the sign-in page (GitHub button only). Claude's connector callback URL (`https://claude.ai/api/mcp/auth_callback`) is allow-listed. **Verify** the exact callback URLs against current Claude connector docs. Access tokens are short-lived (1 h), refresh tokens rotate, both stored hashed.

### 4.5 Engines

- **Anthropic API key (user's own)**: available at launch. Pasted once, written **straight to the user's repo as the Actions secret `ANTHROPIC_API_KEY`** and not kept in OpenPocket's database or logs; `claude-code-action` in their own Actions uses it, like Viber today, with no approval tap. No Claude subscription tokens on hosted (enforced in code: the hosted build rejects `sk-ant-oat…` tokens).
- **GitHub Copilot coding agent**: offered alongside. Checks run on the **Vercel build** (previews), not GitHub Actions, so there is no "Approve and run workflows" tap; GitHub Actions tests exist only for users who switch them on (they then accept the approval tap for Copilot PRs).
- **Own Vercel/Neon tokens** (self-host, and the first hosted beta).

### 4.6 Branch protection and the enhanced-security channel (hosted)

- On hosted, the prod gate is the spoken code **and** GitHub branch protection on the user's `production` branch so that only OpenPocket (the App) can push to it (OpenPocket sets it with the Administration permission and `doctor` re-checks it; if the plan does not allow it, OpenPocket says so in plain words).
- **Enhanced security is a toggle.** On hosted it sends the code by **WhatsApp through a Twilio number** (needs a Meta-approved message template and costs a small amount per message). Self-hosters get **ntfy** by default, or bring their own Twilio.

---

## 5. Opinionated app stack, previews and Neon

### 5.1 The stack (no menu)

- Node (LTS) always. Front end: **Vite** (React by default). Backend: Node functions on Vercel (`/api/*.ts`). One repo, one `package.json`, npm.
- A **starter template** repo (`openpocket-starter`, Vite + `api/health.ts` + test + `vercel.json`) used for "Create a new repo for me". Existing repos are not rewritten: OpenPocket adapts to a repo with a `package.json` and asks before adding `vercel.json`/`api/`.
- Rules written to the app's agent instructions (`copilot-instructions.md`, `AGENTS.md`/`CLAUDE.md`): use the stack, never add Next.js, edit the declarative schema file `db/schema.sql` (never hand-written migration files), no secrets in code, do not deploy.

### 5.2 Database: recommended only when needed

1. Coding runs may mark `needs_database` in the PR description (a fixed marker the rules ask for) or OpenPocket detects imports of `pg`/`@neondatabase/serverless`/`db/schema.sql`.
2. Claude tells the user: "This needs a database. Recommended: Neon, free tier. Say yes."
3. On yes → "Connect Neon" (if not connected) → OpenPocket creates one project `openpocket-<app>` with default branch `main` (production data) in the user's account.
4. Vercel env `DATABASE_URL` is set: production → `main` branch connection; preview → per-preview branch (5.3).

### 5.3 Neon branch per preview

Preferred: the **official Neon–Vercel integration** (branch-per-preview: creates a Neon branch per Vercel preview deployment and injects `DATABASE_URL`/`DATABASE_URL_UNPOOLED` for that deployment; deletes branches when the preview is removed). **Verify** current behaviour (`neon.com/docs/guides/vercel-managed-integration`, `.../vercel-native-integration`).

| Option | Pros | Cons |
|---|---|---|
| A. Neon–Vercel integration installed by the user (Recommended for v1) | Maintained by Neon/Vercel; branch lifecycle tied to deployments; zero OpenPocket code for branches | One more install for the user; OpenPocket cannot fully hide it; OpenPocket depends on its env naming |
| B. OpenPocket manages branches through the Neon API per PR (branch `preview/pr-<n>` from `main`, env var per preview via Vercel API) | Full control, one "Connect Neon" screen, can seed/reset | More code, OpenPocket must clean up (PR closed webhook), Vercel per-branch env var API limits |

**Decided (decision 11): OpenPocket-managed branches (option B), from a seed branch with fake data, never copying real data.** It keeps the "just-in-time, one consent" journey and lets OpenPocket enforce "previews never touch prod data". The spike (A17) only checks that Vercel's per-branch env API is reliable enough; option A stays as the fallback.

Preview data: a branch forked from `main` would be **real data**, so previews are **always** branched from the `preview-seed` branch (schema + fake seed), created at first database setup from the schema file + seed script. Real data is never copied into a preview or staging. (Matches today's rule "previews never use prod data".)

**Schema changes: declarative with Atlas (Decided, decision 13), no versioned migration files.** The agent edits `db/schema.sql` (the desired state). Atlas plans the diff against the target database (`atlas schema apply --dry-run`) and flags destructive changes (dropped tables/columns, narrowed types). OpenPocket reads the plan and says it in plain words ("adds a column `done` to `todos`; loses no data"). Flow:

| Target | When Atlas applies it | Approval |
|---|---|---|
| Preview branch | at preview creation and on every push (by OpenPocket's job, from the schema file) | none |
| Staging branch | when the PR merges to `main` | only if the plan is destructive (as Viber: staging data loss needs approval) |
| Production (`main` Neon branch) | **only after the spoken code**, by OpenPocket right before the promote | always: the plan, its hash and "loses data / doesn't" are read out; the hash is checked before applying |

Atlas is **not** run inside the Vercel build, so a build can never change production data. Atlas runs from OpenPocket's job runner (binary or the Atlas Go library; spike in A20) against the user's Neon connection string.

### 5.4 Preview freshness

No hot reload: every push rebuilds. The task's preview URL is stable per PR (`<project>-git-<branch>-<team>.vercel.app`). OpenPocket injects (via the starter template) a tiny `build-version.json` and a client snippet that polls it every 10 s and shows a "New version ready: refresh" toast (auto-refresh in preview only). OpenPocket also notifies "new build ready" once the Vercel deployment for the new SHA is READY.

### 5.6 Preview, staging and production on Vercel + Neon (Decided: all three, option A, fixed)

| | Preview | Staging | Production |
|---|---|---|---|
| Git | PR branch (`feat/<name>`) | `main` | `production` branch (fast-forwarded from the proven `main` commit by OpenPocket; protected so only OpenPocket can push, 4.6) |
| Vercel | Preview deployment, stable per-branch URL | Preview deployment of `main` with a **branch-assigned domain** `<app>-staging.vercel.app` (or `staging.<custom domain>`, 3.1 step 14d) | Production deployment, production domain (the project's custom domain field, if set) |
| Neon | Branch `preview/pr-<n>` from `preview-seed` (fake data) | **One long-lived branch `staging`** with fake data (same schema as production), **reset only on request** (decision 26) | Branch `main` (the real data) |
| Env vars | Preview scope: `DATABASE_URL` set per branch by OpenPocket | Preview scope **for git branch `main`**: `DATABASE_URL` = staging branch | Production scope: `DATABASE_URL` = `main` branch |
| Created | on PR open, deleted on PR close | at first publish (staging branch with the database, if any) | at first publish |
| Gate | none | none (merge = staging) | spoken 4-digit code, then promote |

**How staging works on Vercel.** Vercel's built-in model is Production + Preview. Two ways to get a third environment:

| Option | Pros | Cons |
|---|---|---|
| A. **Production branch = `production`; staging = `main` as a Preview with a branch domain and branch-scoped env vars** (**Decided, used on every plan**) | Works on every Vercel plan including Hobby; one Vercel project; merge to `main` deploys staging with no extra step; production only moves when OpenPocket advances `production` | A second git branch to keep linear; staging URL is a "preview" in Vercel's UI |
| B. Custom Environment `staging` (Vercel custom environments) | Cleaner in Vercel's UI | Paid plans only; **rejected**: no custom-environments variant exists (decision 25). Changing this means changing the source. |

`promote(task)` fast-forwards `production` to the commit that passed on staging (a deliberate merge, never a force-push), then reports Live when the production deployment is READY. Production never deploys on merge to `main`.

**How a database change travels:** the Atlas plan from `db/schema.sql` (5.3) is applied to the preview branch on push, to the `staging` branch when the PR merges to `main`, and to the `main` branch only after the spoken approval that shows what the change does and whether it loses data. Staging needs approval only if the change drops data.

**Safety rules:** previews and staging never receive production credentials or production data (separate Neon branches, separate env scopes; a check in `doctor` fails if a production `DATABASE_URL` is visible in Preview scope); production deploys of an app are serialized per project; nothing deploys production from a webhook alone.

### 5.5 Claimable Neon: evaluation

Facts ([docs](https://neon.com/docs/reference/claimable-neon)): an agent can create a Postgres project without any Neon account; it **expires after 72 h unless claimed**; limit 100 MB storage and 1 GB transfer; claim codes expire after 15 minutes; **starting a claim revokes the connection credentials** and the app must fetch a new `DATABASE_URL`; identity assertions are secrets with 15-minute bearer tokens.

| | |
|---|---|
| Upside | Zero-account first run: "Make a to-do app" gets a working database with no consent screen. |
| Downside | Data loss if the user does not claim within 72 h (a to-do list entered on day one vanishes); claiming breaks the live connection string; capacity limits; the user must create the Neon account anyway to keep it; extra state to track. Violates "never lose a user's data". |

**Decided (decision 12): Claimable Neon is not used at all for now**: no throwaway demos in v1. Standard path: "Connect Neon" when a task needs a database (a single consent; Neon sign-up with GitHub is one tap). Revisit if Neon extends expiry or offers claim-without-credential-break.

---

## 6. Routing tasks: Copilot vs Claude

`projects.engine` (default per user) and `tasks.engine` (copied at start):

| Engine | When | Who pays | Runs where |
|---|---|---|---|
| `copilot` | Offered on hosted; the user's choice at step 7 | User's Copilot plan (premium requests; free plan limited, Pro recommended) | GitHub's cloud agent; checks on the Vercel build |
| `claude_actions` (API key) | **Available at launch**; hosted user pasted their own Anthropic API key (repo secret only) | User's API usage | User's repo GitHub Actions (`claude-code-action`, `ANTHROPIC_API_KEY`), no approval tap |
| `claude_actions` (subscription token) | **Self-host / Dani's instance only** (`ALLOW_SUBSCRIPTION_TOKEN=true`, refused in hosted mode) | Owner's subscription, personal use | Owner's repo Actions |

### 6.1 Copilot flow

1. `start_task` → OpenPocket creates an **issue** in the user's repo: title, brief and the OpenPocket rules (stack, "do not deploy"), label `openpocket`, task id in a hidden comment.
2. Assigns the issue to Copilot via GraphQL `replaceActorsForAssignable` / `updateIssue` (actor `copilot-swe-agent`) with `agentAssignment` (target repo, `baseRef: main`, custom instructions) and the header `GraphQL-Features: issues_copilot_assignment_api_support`. Uses the **user-to-server token** (refreshed as needed). **Verify** exact input fields at implementation time (preview API).
3. Copilot opens a draft PR (`copilot/…` branch). Webhook `pull_request` links it to the task (issue reference + author `Copilot`). OpenPocket renames the title to `OpenPocket #n: title` as today.
4. **Follow-up** (`continue_task`): OpenPocket comments `@copilot <instructions>` on the PR (comment must come from a user token; Copilot reacts to comments from users with write access).
5. **Checks/preview (Decided):** for Copilot, checks run on the **Vercel build** (typecheck/build as the build command, preview deployment), not GitHub Actions, so there is **no "Approve and run workflows" tap**. GitHub Actions tests are only for users who switch them on; then OpenPocket shows an "Approve checks" link for the Copilot PR (**Verify** the repo setting "Require approval for Copilot").
5a. **Quota hit:** if Copilot refuses or the session reports no quota, OpenPocket tells the user in plain words ("Your Copilot plan has run out for now. Upgrade to Copilot Pro, or add a Claude API key") and pauses the task; nothing is retried silently.
6. **Stop** (`stop_task`): comment "stop", close the PR, unassign. Limitation: Copilot sessions cannot always be cancelled by API; mark Paused in OpenPocket anyway.
7. **Merge/deploy**: unchanged (`deploy_to_prod` merges the PR; Vercel deploys).

Differences to handle: no live step feed (progress hooks are Claude-only; for Copilot read the session log link and PR timeline); branch names are Copilot's, not `feat/…`; no "merge main into branch" step (Copilot starts from the latest base; OpenPocket does the sync via the GitHub update-branch API before merge, as `syncWithMain` does).

### 6.2 Claude (Actions) flow

Existing Viber flow, kept: dispatch `openpocket.yml` (rename of `viber.yml`), `claude-code-action`, progress hooks, `continue_task` on the same branch. Hosted users on API key get the workflow written to their repo by PR; the workflow never contains the key (repo secret).

### 6.3 Routing rules

1. Project engine explicit → use it.
2. Else hosted: the engine the user chose at setup; if Copilot is not assignable on the repo, prompt to add a Claude API key.
3. Else self-host: configured default (`DEFAULT_ENGINE`).
4. Subscription tokens are rejected when `OPENPOCKET_MODE=hosted`. Tests: hosted mode + `sk-ant-oat` token = 400.

---

## 7. OpenPocket's own hosting: Cloud Run + GCP vs Vercel + Neon

Self-hosters should need only GitHub, Vercel, Neon and Claude accounts.

| | A. Stay on Cloud Run/GCP | B. Move to Vercel + Neon (Decided) |
|---|---|---|
| Accounts for a self-hoster | + Google Cloud (billing, `gcloud`, WIF, Artifact Registry) | GitHub, Vercel, Neon only |
| Setup | `deploy.sh`, workload identity: many steps | "Deploy to Vercel" button + env prompts |
| Long requests / state | Container, in-memory prod queue and timers work as-is | Functions are stateless; the in-memory prod queue (`prodqueue.ts`), timers and the periodic sweep need rework |
| MCP streaming | Fine | Streamable HTTP works on Vercel functions; keep within the max duration (Fluid compute allows long durations; **Verify** limit on Hobby/Pro) |
| Background work (sweep, prod batches, `refresh`) | In-process intervals | Vercel Cron (min interval depends on plan; Hobby is once a day, **Verify**), plus event-driven GitHub webhooks, plus a DB-backed job table with lease locking |
| Cost at idle | Min instances 0: near zero | Hobby free; Pro $20/seat if commercial/hosted |
| Cold starts | Container start | Function cold start; Fluid reduces it |
| Ops for Dani | Two clouds (GCP + Vercel for users' apps) | One platform, which also dogfoods OpenPocket's stack |
| Risk | None (works today) | Port effort ≈ L; behavioural risk for prod batches and timers |

**Decided (decision 7): B, Vercel + Neon, for the public OpenPocket (hosted and self-host).** Dani's instance is also Vercel + Neon (decision 17); **Viber stays on Cloud Run until it is retired.** Effort is expected to be **a handful of Viber tasks (steps E1-E2), not 10-15 days**. Done as a re-platforming:

- Keep Express app but export it as a Vercel function (`api/index.ts`), or move to Hono; one runtime-neutral `handler`.
- Replace in-memory queue/timers with a **DB-backed job table** (`jobs`: kind, payload, run_at, locked_until). Prod batch = a job that runs after the window; a cron endpoint + webhook-triggered pokes process due jobs.
- Replace polling `sweep` with webhooks (GitHub App) plus a Cron fallback every minute on Pro (hourly/daily on Hobby: self-hosters on Hobby get the slower fallback; document).
- Neon for OpenPocket's DB (already), Vercel env for secrets (encrypted env; no Secret Manager).
- Previews of OpenPocket's own PRs: Vercel preview + Neon branch (this is the same flow OpenPocket builds for users: dogfooding), replacing `viber-preview-<n>` Cloud Run services and `seed.ts` DB management.
- **Dani's own instance** runs on Vercel + Neon like the hosted one. Viber (Cloud Run) keeps serving him until the cut-over; the Node `server.ts` adapter is kept only until Viber is retired; Cloud Run support and `deploy.sh` are then removed from OpenPocket.

Fallback if the spike (A26) fails: keep Cloud Run for OpenPocket's own server as an "advanced" path; the rest of this plan is unaffected.

---

## 8. Data model changes for multi-user

Schema stays additive and `IF NOT EXISTS`. Single-tenant (Dani, self-host) maps to one implicit user created at start (`users.id = 1`), so nothing breaks.

| Table | Change |
|---|---|
| `users` (new) | `id`, `github_id` (unique), `login`, `name`, `avatar_url`, `email` (nullable), `status` (active/deleting/deleted), `created_at`, `last_seen_at`, `engine_default`. |
| `identities` / `provider_connections` (new) | `id`, `user_id`, `provider` (`github`, `vercel`, `neon`; no `anthropic` row: API keys live only as repo secrets), `external_id` (installation id, team id, org id), `scopes`, `access_token_enc`, `refresh_token_enc`, `expires_at`, `status` (active/revoked/error), `created_at`, `revoked_at`. Tokens encrypted with a per-record data key (section 9). |
| `github_installations` (new) | `installation_id`, `user_id`, `account_login`, `repository_selection`, `suspended_at`. |
| `oauth_clients`, `oauth_codes`, `oauth_tokens` (new) | MCP authorization server state: client id/secret (DCR), redirect URIs, code + PKCE challenge, hashed access/refresh tokens, `user_id`, `expires_at`, `revoked_at`. |
| `projects` | + `user_id` (NOT NULL for new rows; backfill to the owner user), + `installation_id`, `engine`, `vercel_project_id`, `vercel_team_id`, `neon_project_id`, `neon_preview_seed_branch`, `neon_staging_branch`, **`custom_domain`** (nullable), `custom_domain_state` (pending/verified/error), `staging_subdomain` (nullable), `production_branch` (default `production`). Unique `(user_id, repo_owner, repo_name)`. The environment set is fixed, so there is no per-project environment config. |
| `usage` (new) | `user_id`, `month`, `tasks_started`: powers the hosted free-tier cap (default **100 tasks/month per user**, adjustable in `instance_settings`). |
| `tasks`, `task_rounds`, `progress_events`, `schema_waits`, `activity` | + `user_id` (denormalised, indexed) so every query filters by user; `tasks.engine`, `tasks.copilot_issue`, `tasks.neon_branch`. |
| `codes` (deploy codes) | + `user_id`; code bound to user, task, commit, hour as today. |
| `settings` (existing global) | Split: `instance_settings` (owner config) and `user_settings` (notify channel, enhanced security, owner name). |
| `notify_channels` (new) | per user: ntfy topic (generated, secret), optional email, optional WhatsApp number (for the enhanced-security code via Twilio, hosted); replaces global `NTFY_TOPIC` for hosted. |
| `audit_log` (new) | `user_id`, `actor`, `action` (connect/disconnect/deploy/delete/token_use), `detail` (no secrets), `at`. |
| `jobs` (new, section 7) | durable queue for prod batches, refresh, cleanup, deletion. |
| `webhook_deliveries` (new) | `delivery_id` unique, for idempotency. |

Rules:
- **Every query goes through a `scope(userId)` helper**; no raw table access in route handlers. Tests: "user A cannot read, start, merge or deploy user B's task/project" for each MCP tool and HTTP route.
- Postgres **row-level security** as a second layer (`SET app.user_id` per request, policies on the user-owned tables). Recommended but start with the helper; add RLS in step E2.
- The prod queue is per (user, project) instead of per project; batch rule unchanged.
- Task numbers: per-user sequence (`user_task_seq`) so the "#n" each user hears starts at 1; internal id stays global.
- Dani's instance: `OPENPOCKET_MODE=single` hides sign-in and uses admin auth; the same tables, one user. It is a **separate deployment and database** from the hosted one, not a tenant of it.

---

## 9. Security

| Area | Plan |
|---|---|
| **Tokens at rest** | AES-256-GCM, envelope encryption: one **key-encryption key** (`ENCRYPTION_KEY` / KMS-held in hosted) wraps a per-record data key; ciphertext has a key-version prefix for rotation. Tokens never in logs, errors or task history (`redact.ts` extended with Vercel/Neon/GitHub App patterns). Never returned from any API or MCP tool. |
| **Sessions / MCP tokens** | OpenPocket access tokens 1 h, refresh tokens rotating with reuse detection, stored hashed (SHA-256). Web session cookie `HttpOnly; Secure; SameSite=Lax`. CSRF protection on state-changing routes. |
| **Least privilege** | GitHub App: only selected repos, no Administration; Workflows/Secrets permissions only for the Claude engine. Neon/Vercel: smallest scopes (4.2, 4.3). OpenPocket acts only on repos in `projects` for that user. |
| **Revocation** | *User-side:* Account page "Disconnect" per provider: calls the provider's revoke endpoint where one exists (GitHub App user token revoke, Vercel token revoke, Neon token revoke), deletes the stored tokens, marks connection `revoked`, pauses tasks needing it. *Provider-side:* GitHub webhook `github_app_authorization` (revoked), `installation deleted`, Vercel/Neon 401 → connection `revoked` and a notification "Reconnect GitHub". *OpenPocket-side:* admin can revoke all tokens of a user (kill switch) and rotate the KEK. |
| **Deleting a user** | "Delete my account" (web, MCP asks to confirm in the web UI): 1) revoke and delete all provider tokens, 2) stop and pause running tasks, 3) delete rows in `users`, `identities`, `tasks`, `task_rounds`, `progress_events`, `oauth_*`, `notify_channels`, `codes` (cascade by `user_id`, in a job, idempotent), 4) keep a minimal tombstone (`github_id` hash, deleted_at) for abuse handling; 5) email/ntfy confirmation. **Not touched:** the user's repos, Vercel projects and Neon databases (they own them); the UI lists them with links so they can remove them. Completed within 30 days (target immediate), documented in a privacy policy. Backups age out (Neon PITR window). |
| **Anthropic API keys** | Stored **only as a GitHub repo secret** in the user's repo (write-only Secrets permission); never in OpenPocket's database, logs or memory beyond the single request that writes it. Removing the project or the key deletes the secret. |
| **Data held by OpenPocket (hosted)** | Account (GitHub id/login/avatar, optional email), task briefs and summaries, PR metadata/URLs, encrypted provider tokens, audit log, usage counts. **Not held:** repo source code (only transient), app data, app databases, users' deploy secrets. **Hosted legal entity, privacy policy, ToS and GDPR are parked** (decision 6): not needed for the open-source publish, **required (with a lawyer review) before opening hosted sign-ups.** |
| **Prompt injection / abuse** | Repo content and issue text are untrusted: engines run with repo-scoped permissions only; OpenPocket never executes repo code itself; briefs cap length; per-user rate limits (tasks/hour, deploys/hour); per-user cap on open tasks. |
| **Prod safety** | Unchanged and mandatory: first `deploy_to_prod` call never merges; code bound to user/task/commit/hour; enhanced-security option; DB-change approval. Production only moves when OpenPocket advances the `production` branch (merge to `main` deploys staging only). On hosted the gate is **the spoken code AND branch protection** on the user's `production` branch so only OpenPocket can push (4.6, decision 14). Enhanced security is a toggle: hosted sends the code by WhatsApp (Twilio, approved template), self-hosters by ntfy or their own Twilio. |
| **Tenant isolation** | Section 8 scoping + RLS; MCP tokens bound to user; webhooks verified by HMAC and resolved to the user via `installation_id`. |
| **Webhooks/callbacks** | HMAC verification, replay protection (`delivery_id`), OAuth `state` + PKCE, strict redirect URI allow-lists. |
| **Supply chain** | Dependabot, gitleaks in CI, lockfile, minimal dependencies, pinned actions by SHA. |
| **Subscription-token ban** | Hosted code path rejects subscription-shaped tokens and the setting cannot be enabled by env in hosted mode (checked at boot: `OPENPOCKET_MODE=hosted` + `ALLOW_SUBSCRIPTION_TOKEN` → refuse to start). |
| **Incident** | `SECURITY.md`, private advisories, KEK rotation runbook, "revoke all" switch. |

---

## 10. Cost to Dani of running the hosted OpenPocket

Estimates (Nov 2026 list prices, **Verify** before launch); users bring their own Vercel, Neon, Copilot, so these are only OpenPocket's own infrastructure.

| Item | Cost / month | Notes |
|---|---|---|
| Vercel (OpenPocket server) | $0 (Hobby, not allowed for a commercial service) → **$20 per seat** on Pro + usage | Hosted service is a product: assume Pro. Fluid compute usage small (MCP calls are short): $0-10. |
| Neon (OpenPocket DB) | $0-19 | Free tier OK for early; Launch ($19 plan) when scale/uptime needed. Rows per user are small (<1 MB each). |
| Domain | ~$1-2 (about $15/year) | |
| Email (Resend) / push (ntfy.sh) | $0 | Free tiers; ntfy per-user topic. |
| WhatsApp codes (Twilio) | number ~$1-2 + a few cents per message | Only for users who turn enhanced security on; needs an approved template. |
| GitHub App, OAuth apps, Copilot API | $0 | Users pay Copilot. |
| Monitoring/logs | $0-10 | Vercel logs + an uptime check. |
| Support / abuse handling | time | The real cost; set expectations ("as is" in v1). |

**Total: roughly $20-60/month at launch, flat until hundreds of active users;** scales mostly with function invocations from webhooks and sweeps. At 1,000 users with ~10 tasks/month each, expect low hundreds of dollars. Cost guards: per-user rate limits, jobs with backoff, no polling for idle users (webhook-driven), idle-project sweeps at a lower rate. Free tier: a generous cap, e.g. **100 tasks/month per user**, adjustable (decision 20). Not in the cost: Dani's own Claude subscription (personal instance only) and users' own bills.

---
## 11. Secrets and privacy audit (of the current Viber tree and history)

Still valid and unchanged: the OpenPocket repo is created from a scrubbed export of this tree, so these findings are the scrub list. "Viber" in the findings means the current private repo.

### What was scanned

- **Current tree** (`git ls-files`, 108 files) and **all history**: 175 commits, all branches/refs in this clone, every added line (`git log --all -p -G`), plus every path ever present (including deleted ones) and all commit messages.
- Patterns: GitHub tokens (`ghp_`, `github_pat_`, `ghs_`), Anthropic keys (`sk-ant-`), AWS/Google keys (`AKIA`, `AIza`), private-key blocks, Slack tokens, Neon (`npg_`), `scheme://user:pass@host` URLs, `.env`/key/credential/`.pem` file names, GCP numeric project IDs, `*.run.app`/`*.neon.tech` hosts, ntfy topics (`viber-` + 24 hex), and personal/project terms (`kupferco`, `dnkupfer`, Dani, email domains, `terapay`, `lv-notas`, `housekeeper`, `pond`, `therap*`, `patient`).
- Tool: plain `git grep`/`git log -G` only. No dedicated scanner (gitleaks/trufflehog) was available offline.

### Could not be scanned

- **GitHub-side data** is not in the git history and was not reachable: Actions logs and artifacts, PR and issue bodies/comments, task briefs (they live in the Viber database and in PR descriptions), repo/Actions secrets, Cloud Run env, Secret Manager, the Neon database. All of it stays private by construction (the fresh repo does not carry it) but should be reviewed once by hand (see "Manual checks").
- Binary screenshots (`docs/screenshots/*.png`, 20 files) were not inspected pixel by pixel. They are produced by `ui:check` from seeded fake data, but a person must look at them before publishing.
- Other repos (Housekeeper, TeraPay, ...) were out of scope.

### Findings

**No live secret was found in the tree or in history** (no token, key, private key, password or credentialed URL). Details:

| # | Finding | Where | Severity | Action |
|---|---|---|---|---|
| F1 | No real credentials. The only credential-shaped strings are placeholders and test fixtures (`postgres://user:pass@ep-xxx…`, `postgres://u:p@localhost/db`, `viber:viber@localhost` for the CI test DB) and redaction regexes (`ghp_`, `sk-ant-`, `AKIA`, `xox`) in `redact.ts`. | `.env.example`, `*.test.ts`, `.github/workflows/deploy.yml`, `redact.ts` | None | Keep. Fresh repo makes this moot anyway. |
| F2 | Deleted code mentioned a "Routine token (sk-ant-oat01-…)" placeholder input and `routines.ts` (the dropped Routines approach). Only a placeholder prefix, no value. | history only | Low | Gone with a fresh repo. |
| F3 | Owner's **personal email** `daniel@kupfer.co` and sender `noreply@kupfer.co` as defaults. | `.env.example:15-16`, `apps/api/src/config.ts:64`, `scripts/deploy.sh:246` | Medium (personal data) | Replace with empty/required values (section 12). |
| F4 | **Commit author identities**: `Daniel Kupfer <daniel@kupfer.co>` (9), `Kupfer <studio@kupfer.co>` (62), bot and `noreply@anthropic.com` (102). | git history | Medium | Not carried over (fresh repo). **Decided (decision 3):** the first public commit and all later ones use the real name **Daniel Kupfer** with the **GitHub no-reply email** (`<id>+kupferco@users.noreply.github.com`), set in `git config` of `openpocket-private` and the publish script. |
| F5 | **Another GCP project id** `shopping-450413` (Housekeeper's), plus service account `housekeeper-deployer@shopping-450413.iam.gserviceaccount.com` and bucket `housekeeper-evidence-staging`, used as realistic fixtures. | `failure.test.ts`, `manual.test.ts`, `manual.ts` (comment), `scripts/deploy.sh` (6 commits) | Medium (infra identifiers of a real project; not a secret but should not be public) | Replace with `example-project-123` / `myapp-deployer@…`. |
| F6 | **Own infra identifiers**: GCP project `viber-kupferco`, region `europe-west2`, repo `kupferco/viber`, host pattern `viber-xxxxx.europe-west2.run.app`. | `deploy.yml:271-272`, `deploy.sh:17-25`, `templates/deploy-cloudrun.yml:41`, `templates/preview-callback.yml:5`, `onboard.ts:287,290`, `README.md` | Low (identifiers, no access by themselves) | Parameterise (section 12). |
| F7 | **Real project/customer names** used as examples: Housekeeper, TeraPay (`kupferco/terapay`, `lv-notas`), a TeraPay-specific line in the README test plan, and a CLAUDE.md decision "TeraPay is out of this for now. Real service, real people". Also "patient data" wording in `repos/requests.ts` and `testuser.ts` (generic wording, no data). | `README.md:46,135,197`, `CLAUDE.md:230`, `mcp.ts:104,118,162`, `App.tsx:1094-1095`, tests | Medium (business context; no patient/therapist data was found anywhere) | Use neutral examples (`acme/webapp`, "Webshop"). Drop TeraPay lines. |
| F8 | **Personal name in product text** ("Dani's phone", "ask Dani to read it out", "waiting for Dani") in ~25 code comments and **user-facing strings** (service.ts:1040, 1057, 1073; mcp.ts:91). | `apps/api/src/*`, `apps/web/src`, `CLAUDE.md` (9×) | Low | Replace with "the owner" / configurable `OWNER_NAME`. |
| F9 | **ntfy topic**: the code carries no topic. It is generated at setup (`viber-` + 24 hex) and also carries deploy confirmation codes. Not in history (0 matches), but exists in Secret Manager and on Dani's phone. | `deploy.sh:206` | Medium if ever leaked | Not in the repo. **Rotate when switching instances anyway** (below). |
| F10 | `README.md` and `CLAUDE.md` describe the private deployment in detail (workload identity pool conditions, Environment `production` with Dani as reviewer, Secret Manager names). | `README.md:146-174`, `CLAUDE.md` | Low | Rewrite for self-hosters (section 13). |
| F11 | **The runner's `origin` URL embeds a short-lived `ghs_…` installation token** (`.git/config` of this checkout). Not in history, expires within about an hour, but anyone who copies the `.git` directory of a CI checkout leaks it. | local checkout only | Low | Never zip/upload `.git` from a runner. The scan of the public repo must run on a clone made without credentials. |
| F12 | History contains 93 `claude[bot]` commits whose **messages quote briefs** (e.g. "Onboarding sets VIBER_PUSH_TOKEN…", "Strip whitespace from pasted tokens"). No secret values, but they expose private workflow details and task numbering ("Viber #n"). | git history | Low | Fresh repo drops them. |

### Rotate regardless

Nothing was found leaked, so nothing *must* be rotated because of the audit. Rotate at the **migration cut-over** (section 15), because the private repo keeps living and its Actions logs/briefs are not public-safe:

| Secret | Why |
|---|---|
| ntfy topic (`viber-ntfy-topic`) | Topic is also the channel for deploy codes; new instance gets a new one. |
| `MCP_SECRET`, `HOOK_SECRET` | New deployment, new URL; old connector path stops working. Hook secret is copied into every app repo by re-sync. |
| `GITHUB_TOKEN` (fine-grained PAT) | New token scoped to the public repo plus app repos; revoke the old one after cut-over. |
| `CLAUDE_OAUTH_TOKEN` | Re-run `claude setup-token` for the new instance, so the private and public deployments never share it. |
| `ENCRYPTION_KEY`, `ADMIN_PASSWORD` | Changing logs out sessions; per-repo preview test passwords derive from `ENCRYPTION_KEY`, so re-sync apps afterwards. |
| Neon database credentials | New role/branch for the new instance (or keep the DB and rotate the password). |

### Manual checks before the first public push

1. Run `gitleaks detect --no-git` (and later `gitleaks git`) and `trufflehog filesystem .` on a clean clone of the **new** repo's initial tree. Keep gitleaks as a CI job afterwards.
2. Open each `docs/screenshots/*.png` and confirm only seeded fake data shows (no real project names, repo names or emails).
3. Search private Actions logs and PR descriptions for anything that was copied into code comments or tests (the `shopping-450413` fixtures suggest real logs were pasted into tests once).
4. Confirm no `.env` is on any developer machine path that gets copied into the new repo; `.gitignore` already excludes it.


---

## 12. Hardcoded to Dani, and a single config approach

Updates for OpenPocket: OpenPocket runs on Vercel + Neon (decision 7), so GCP items only matter for Viber until it is retired; the config shrinks to `PUBLIC_URL`, `DATABASE_URL`, tokens and owner details. Rows about GCP/Secret Manager/WIF are then deleted instead of parameterised.

### Proposal (Recommended)

One place for each kind of setting:

- **`.env` / env vars** (secrets and per-instance values): documented in `.env.example`, validated in `config.ts` (already the pattern; it throws on a missing required value).
- **`viber.config.json`** (non-secret, committed in the self-hoster's fork): `owner`, `gcpProject`, `region`, `serviceName`, `repo`, `notifyFrom`, `ownerName`. `deploy.sh`, the workflows and the API all read it, so there is one file to edit. Workflows read it with `jq` in a first step; GitHub Actions repo **variables** (`vars.*`) can override.
- A **setup script** (`npm run setup`, section 3.2) writes both files and sets the GitHub variables, so nobody edits YAML by hand.

### Inventory

| Item | Today | Where | Proposed setting |
|---|---|---|---|
| GitHub owner/repo of Viber | `kupferco/viber` | `deploy.sh:25`, `onboard.ts:287,290` (`uses: kupferco/viber/.github/actions/report@main`), `deploy.yml` trust condition, README | `VIBER_REPO` (default: from `GITHUB_REPOSITORY`). **Important**: onboarding writes `uses: <upstream>/…/report@main` into every app. It must point at the self-hoster's own fork, not upstream (also decision 7). |
| GitHub owner of apps | `kupferco`, `dnkupfer` (token scope, examples) | README, UI placeholder, MCP descriptions | Derived from the token's user; remove from examples. |
| GCP project | `viber-kupferco` | `deploy.sh:22`, `deploy.yml:271` | `GCP_PROJECT` (config / repo var `GCP_PROJECT`) |
| Region | `europe-west2` | `deploy.sh:23`, `deploy.yml:272`, `templates/deploy-cloudrun.yml:41` | `GCP_REGION` |
| Cloud Run service names | `viber`, `viber-staging`, `viber-preview`, `viber-preview-<n>` | `deploy.sh`, README, `deploy.yml` | Prefix `SERVICE_PREFIX` (default `viber`). |
| Workload identity | pool `github`, provider/SA bound to `kupferco/viber` | `deploy.sh:104-141`, `vars.GCP_WIF_PROVIDER`, `vars.GCP_DEPLOY_SA` | Already variables; script binds to the fork's `GITHUB_REPOSITORY`. |
| Secret Manager names | `viber-ntfy-topic`, `viber-database-url-*`, `viber-preview-test-password` | `deploy.sh`, `deploy.yml` | Keep (they are namespaced per project); prefix from `SERVICE_PREFIX`. |
| Neon | one server, DBs `viber`, `viber_viber_staging`, `viber_preview_<n>` | `.env.example`, `config.ts`, `previewdb.ts` | `DATABASE_URL` (exists). Document that previews need a second DB on the same server. |
| ntfy | topic generated; server `https://ntfy.sh` | `deploy.sh:206`, `config.ts:65-68` | Already `NTFY_TOPIC`, `NTFY_SERVER`, `NTFY_TOKEN`. No change. |
| Email | `daniel@kupfer.co`, `Viber <noreply@kupfer.co>` | `.env.example`, `config.ts:64`, `deploy.sh:246` | `NOTIFY_EMAIL`, `NOTIFY_FROM`: **no default**; email off unless set. |
| Owner name | "Dani" in strings | service.ts, mcp.ts, comments, onboarding CLAUDE.md block (`onboard.ts:326`) | `OWNER_NAME` (default "the owner"). The text written into every app's CLAUDE.md uses it. |
| Public URL / domain | Cloud Run URL, `PUBLIC_URL` | `config.ts` | `PUBLIC_URL` (exists). No custom domain assumed. |
| Preview URLs | `<app>-preview-<n>` on Cloud Run | README, templates | Derived from `GCP_PROJECT`/`GCP_REGION`; no change otherwise. |
| Project list | Housekeeper, Pond, TeraPay in DB; Housekeeper as a legacy marker in `onboard.ts:346` ("first Housekeeper section predates the markers") | DB (not in repo), `onboard.ts` | Projects live in the DB and are not in the repo. Remove the Housekeeper migration branch from the public code (keep it in Dani's private notes). |
| Test fixtures | `shopping-450413`, `housekeeper-*` | tests | Neutral fixtures (F5). |
| README/CLAUDE.md prose | Dani, kupferco, TeraPay | docs | Rewrite (section 13). |
| `CLAUDE.md` Dani-specific rules | "Working with Dani", personal decisions | `CLAUDE.md` | Split: generic dev notes public; personal rules in private overlay (section 13). |

Compatibility note for the later SaaS: per-tenant values (`owner`, tokens, notify channel) should be read through `config.ts` accessors rather than module-level constants, so they can become per-tenant later. No SaaS work is planned here.

### Guard for decision 7 (never open PRs upstream)

- Onboarding and all GitHub writes already act only on repos in the user's `projects` table, using the user's token. Add a **hard allow-list**: refuse any write to a repo whose owner differs from the token's user/orgs listed in `ALLOWED_OWNERS`, and refuse the upstream repo (`kupferco/viber`'s public name) explicitly unless `ALLOW_UPSTREAM=true`.
- Add a test: "a task on the Viber project of a fork never targets the upstream remote".
- Viber working on *itself* (the fork) opens PRs only within the fork.
- Contribution upstream stays a manual `gh pr create --repo upstream` by a human.


Additional hardcoded items found by the new decisions:

| Item | Change |
|---|---|
| Names "Viber", `viber.yml`, `viber-*` service/secret/branch names, `Viber #n` PR titles, `VIBER_*` env vars | Rename to OpenPocket (`openpocket.yml`, `OPENPOCKET_*`, `OpenPocket #n`). Accept the old `VIBER_*` names for one release on Dani's instance. `HOOK_RULES` text in `onboard.ts` is rewritten. |
| `uses: kupferco/viber/.github/actions/report@main` in app repos | Reference `<OPENPOCKET_REPO>` (the instance's own fork); for hosted users no Actions action is needed (webhooks replace it). |
| Claude OAuth token required in config | Optional; engine-dependent (section 6). |
| Single global `ntfy` topic | Per user (section 8). |

---

## 13. What goes into the public OpenPocket repo

| Path | Decision | Notes |
|---|---|---|
| `apps/api`, `apps/web`, `package*.json`, `.gitignore`, `.dockerignore` | **Public** | After scrubs (sections 11-12) and rename. |
| `Dockerfile`, `scripts/deploy.sh`, GCP parts of `deploy.yml` | **Dropped / private** | OpenPocket's own hosting is Vercel + Neon (decision 7, 17); these stay in the private Viber repo until it is retired. |
| `.github/actions/report` | **Public** (self-host Claude engine) | Reference the user's own fork. |
| `.github/workflows/` (CI, openpocket.yml template, deploy for Vercel-based self-host) | **Public, parameterised** | Fork-safe (below). |
| `templates/` | **Public, rewritten** | Starter Vite app (`openpocket-starter`), `openpocket.yml`, rules files. Cloud Run templates dropped. |
| `docs/screenshots/` | **Public after manual review** | |
| `README.md`, `docs/self-hosting.md` | **Public, rewritten** | |
| `CLAUDE.md` | **Split** | Public: shape, commands, generic decisions. Private overlay: "Working with Dani", TeraPay, personal approval rules. |
| `docs/saas-plan.md` | **Dropped**: merged into this plan and superseded (decision 18) | Nothing from it is lost; the multi-user parts are sections 4, 8 and 9. |
| This file | **Private** | Strategy and audit. A trimmed public design doc is written instead. |
| History, briefs, `task_rounds`, logs, PR bodies, Neon data, secrets | **Never public** | |
| New: `LICENSE` (MIT), `CONTRIBUTING.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`, issue/PR templates, `CODEOWNERS`, `.env.example` | **Add** | `PRIVACY.md`/`TERMS.md` come later, with the hosted launch. |

**How to make the fresh repo** (Recommended): prepared branch in the private repo → `git archive` → new directory → `git init` → one initial commit with the chosen public identity → push to `kupferco/openpocket`. Never `git filter-repo` the old repo (unreachable objects, cached PR refs, forks).

### Public repo setup

| Item | Plan |
|---|---|
| `LICENSE` | MIT, copyright holder **Daniel Kupfer personally** (decision 4): `Copyright (c) 2026 Daniel Kupfer`. |
| `README.md` | What Viber is; the **voice-first loop** (say a task → coding run on your Claude subscription in GitHub Actions → PR → preview → "deploy to prod" with spoken code + GitHub approval); screenshots placeholder (`docs/screenshots/`, shown after the manual review); **a clear Claude terms notice** (Claude subscription tokens are for **personal use on your own instance only**; sharing them or running a service on them is not allowed; the hosted OpenPocket uses Copilot or API keys); **safety model for prod** (never one unconfirmed command away: first call describes and returns a code, second call with the code merges; optional enhanced security sends the code only to the phone; GitHub Environment approval; database changes need approval; branch/PR-only coding runs); quick start linking the guide; "this runs your code with your credentials" warning. |
| `CONTRIBUTING.md` | **Issue first**: open an issue and get a maintainer "go" before large PRs. Small fixes and docs may go straight to PR. PR rules: one concern, tests, `npm run typecheck && npm test` pass, no secrets. **No CLA and no DCO: inbound = outbound MIT** (decision 21). **Support is "as is, best effort", via GitHub issues, with no response-time promise** (decision 20). **Dani reviews and merges every PR.** **AI-generated PRs**: allowed if the author has read, understood and tested the change, discloses it, and a human answers review comments; low-effort or unverified bulk AI PRs are closed. Contributions are made by humans: a self-hosted Viber never opens them (decision 7). |
| `SECURITY.md` | Private reporting via GitHub Security Advisories (enable it), no public issues for vulnerabilities, contact via a **dedicated alias** (e.g. `security@<his domain>`) that forwards to Dani (decision 5), best-effort response, supported versions = `main`. Include the threat notes: the instance holds a GitHub PAT and Claude token; MCP path secret; hooks use a bearer secret. |
| `CODE_OF_CONDUCT.md` | Contributor Covenant 2.1 with the same dedicated alias as contact (decision 5). |
| Issue/PR templates | Bug (version, how deployed, logs with secrets redacted), feature (problem first), setup question → Discussions; PR template with checklist (issue linked, tests, no secrets, AI disclosure). |
| Branch protection on `main` | Require PR, require status check `check`, require 1 review by Dani via CODEOWNERS (`* @owner`), no force-push, no deletion, require linear history or merge commit as today, include administrators off only for Dani's emergencies. Restrict who can push to Dani. The repo is `kupferco/openpocket`, a personal repo, so "only Dani pushes" is the default. |
| CI for fork PRs without secrets | Workflows for `pull_request` (not `pull_request_target`) run typecheck, tests, build and gitleaks. They use no secrets and `permissions: contents: read`. All deploy and preview jobs already require `vars.GCP_*` and a same-repo head (`head.repo.full_name == github.repository`); keep that and add `if: github.repository == '<upstream>'`-free logic (gate on variables so forks self-skip). Set "Require approval for all outside collaborators" for first-time contributors' workflow runs. Never check out fork code in a workflow that has secrets. |
| Repo settings | Disable Actions for forks' secrets (default), enable secret scanning + push protection, Dependabot, private vulnerability reporting, Discussions. Visibility public only after section 1 manual checks pass. |

The **hosted** OpenPocket adds `PRIVACY.md`, `TERMS.md`, a status/contact page and the GitHub App's public listing text **before sign-ups open, not before the open-source publish** (decision 6; lawyer review needed). Remove `PRIVACY/TERMS` from the publish checklist (B4). Visibility goes public only after the manual checks in section 11 pass.

---

## 14. How Dani keeps working on OpenPocket without leaking private data

Problem unchanged: Viber tasks put briefs into dispatches, PR bodies and Actions logs; on a public repo these are world-readable.

| Option | Verdict |
|---|---|
| **A. Private mirror (Recommended)**: `kupferco/openpocket-private` is a clone of public `openpocket` (`upstream` remote) plus a small overlay (personal `CLAUDE.private.md`, his config). Dani's instance works on the private repo; the reviewed change is published to public by a script as a squashed diff with a neutral message; public `main` is merged back. | Nothing private on public; Viber works unchanged; matches decision 4. Cost: two repos, one sync script. |
| B. Private fork of a public repo | Not possible on GitHub. |
| C. Work directly on public with redacted briefs | One slip leaks; not recommended. |
| D. Private is the source of truth, public a release mirror | Contradicts decision 3. |

**Flow:** task runs in `openpocket-private` (briefs, previews, logs private) → Dani reviews preview → `scripts/publish.sh <task>` creates a branch in public with the squashed diff excluding `overlay/`, neutral commit message (no brief, no task number) → Dani opens and merges that PR → private `main` fast-forwards from public. Contributor PRs merge on public and flow back down. A CI check on public rejects `overlay/` paths. Nothing automated posts upstream.

**Where the old repo fits:** `kupferco/viber` stays private and keeps running Dani's production instance until cut-over (section 15), then is archived as the history store. `openpocket-private` is the long-term home of his instance. (Alternative: rename `viber` itself to become the private mirror; its history is private anyway. Simpler, but its history contains briefs and the pre-scrub tree, so any later accidental publish of it is a leak: **not recommended**.)

### 14.1 Preview, staging and production for OpenPocket's own development (Decided: all three)

| Environment | OpenPocket's own code, Dani's flow |
|---|---|
| **Preview** | One per task PR in `openpocket-private` (today's `viber-preview-<n>` pattern): own service, own Neon database `openpocket_preview_<n>` seeded with fake data, no GitHub writes and no notifications (`OPENPOCKET_ENV=preview`), deleted when the PR closes. The **public repo has no previews with secrets**: PRs from forks run CI only (section 13); Dani can open a preview of a contributor's change by importing it into `openpocket-private` first. |
| **Staging** | `openpocket-private` `main` deploys to `openpocket-staging` (own database, a test GitHub App registration, test Vercel/Neon OAuth apps; read-only against real repos as `viber-staging` is today). Every change published from public (merged contributor PRs) lands here first, because it flows down into `openpocket-private` `main`. **Decided (decision 27): two separate deployments, each with its own staging and its own production**: (a) Dani's personal OpenPocket: `personal-staging` + `personal-prod`; (b) public hosted OpenPocket: `hosted-staging` + `hosted-prod` (Dani only has a test account there). Same code, separate Vercel projects, Neon databases, GitHub Apps/OAuth registrations and secrets. Staging for the hosted one runs multi mode, the personal one single mode. |
| **Production** | Two production deployments from the same commit, each promoted by hand with the spoken code: (1) **hosted OpenPocket** for standard users (multi-user, Copilot and API-key engines only), (2) **Dani's personal instance** (single-tenant, his Claude subscription token, his projects). Promotion is Viber's existing gated flow (code, batch, GitHub approval), until Viber is retired; afterwards OpenPocket's own promote flow. A release is a tag on the public repo after production is proven; the public repo itself is never deployed from. |

Mapping of repos to environments:

```
fork PR ──▶ public openpocket (CI only, no secrets) ──merge by Dani──▶ public main
                                                                      │ flows down (sync)
Dani task ─▶ openpocket-private PR ─▶ PREVIEW (per PR)                    ▼
                    │ reviewed, then publish.sh ──▶ public PR ──▶ public main
                    ▼ merge
          openpocket-private main ─▶ STAGING ─▶ (spoken code) ─▶ PRODUCTION: hosted OpenPocket
                                                          └▶ PRODUCTION: Dani's instance
```

```
promote(code) ─▶ personal-prod      (single tenant, his subscription)
              └▶ hosted-prod        (multi-user; later, after open-source publish)
staging:  personal-staging, hosted-staging (each own DB and registrations)
```

Order that matters: a change reaches public `main` only after Dani has seen its preview; it reaches production only after it ran on staging. The self-hoster's own fork repeats the same three environments on their own accounts (3.2).

---

## 15. Migration from Viber to OpenPocket (Dani's instance)

Goal: no downtime for current projects and tasks. **Decided (decisions 16, 17, 19):** Dani migrates his projects to **his own self-hosted OpenPocket** (Vercel + Neon, `openpocket-private`) and **retires Viber**, **before the hosted launch**. TeraPay's pipelines are deferred until OpenPocket is running. Two migrations: **code** (Viber → public OpenPocket + private mirror) and **platform** (Cloud Run → Vercel + Neon, now part of the plan, not optional).

### 15.1 Code and name

| Step | Detail |
|---|---|
| 1 | Freeze: finish or pause running tasks (`list_tasks active_only`); nothing in `prod_queued`. |
| 2 | Do the scrub and rename in Viber tasks on `kupferco/viber` (checklist phases 1-2). Dani's instance keeps working during this: accept `VIBER_*` aliases, keep `viber.yml` readable by the new code. |
| 3 | Create public `kupferco/openpocket` from the scrubbed export (steps B1-B3). Create `openpocket-private` from it with the overlay. |
| 4 | Create Dani's **personal OpenPocket on Vercel + Neon** from `openpocket-private`: own Vercel projects (`personal-staging`, `personal-prod`), a new Neon project seeded from a **copy** of Viber's database (additive, `IF NOT EXISTS` schema, so tasks, rounds and projects carry over). Viber keeps running untouched on Cloud Run meanwhile. |
| 5 | Deploy staging/preview from `openpocket-private`, test a task end to end. |
| 6 | Rotate secrets (section 11 table). Update the Claude connector URL if `MCP_SECRET` changes. |
| 7 | Re-sync each app (`onboard` is idempotent): updates workflow (`viber.yml` → `openpocket.yml`; old file removed), CLAUDE.md block, hook secret, report action reference. Housekeeper first; TeraPay deliberately later. |
| 8 | Promote `personal-prod` (spoken code). Switch the Claude connector to the new URL. Existing `projects.repo_owner/name` unchanged; open PRs in the old repo finish there. Task numbers continue in the DB. |
| 9 | **Retire Viber**: archive `kupferco/viber` (read-only history store) or make it private, remove its Cloud Run services and workload-identity binding; revoke old PAT/token. Other old `kupferco` repos are tidied (private/archived) closer to the hosted launch. |

Rollback until step 9: Viber on Cloud Run and its repo are untouched; the connector can point back to it.

### 15.2 Multi-user schema

Run additive migrations (section 8) on Dani's DB: create `users` row 1 for Dani, backfill `user_id` on existing rows, keep `OPENPOCKET_MODE=single`. No data is moved.

### 15.3 Platform

Included in 15.1 step 4: the personal OpenPocket runs on Vercel + Neon from the start, in parallel with Viber behind a separate connector URL, then the connector is switched. Prod batches in flight must be empty at the switch. Cloud Run is removed at retirement (D1/D2).

---

## 16. Ordered build plan (checklist)

Do the steps **in this order**. Each step is one Viber task (S ≤ half a day, M ≈ 1 day, L ≈ 2-3 days). A step starts only when the steps in "After" are live. Stage A runs as Viber tasks on `kupferco/viber` until the private mirror exists; from step B1 the tasks run on `openpocket-private`. Every step ships through preview → staging → production as in 2.1; "Dani" marks a manual action only he can do. **Order of the whole plan (decisions 16-19): build in private → publish open source → move Dani's instance and retire Viber → hosted launch later.**

### Stage 0: decide and prepare (before any task)

| Step | Task | Effort | After |
|---|---|---|---|
| 0.1 | Dani: **apply now** for the Vercel integration listing and the Neon OAuth/partner app (decision 24); trademark check for "OpenPocket"; buy the domain (with suffix); create the `security@` alias forwarding to him; set the git identity (Daniel Kupfer + GitHub no-reply) | S, Dani | |
| 0.2 | Run gitleaks/trufflehog on tree and history, review screenshots; rotate section 11 secrets | S, Dani | |
| 0.3 | ~~Mark `docs/saas-plan.md` superseded~~ Done: its content is merged in Appendix A and it is not part of this repo (decision 18) | done | |

### Stage A: build OpenPocket in the private repo (Viber tasks)

| Step | Task | Effort | After |
|---|---|---|---|
| A1 | Scrub personal data: emails, names, `NOTIFY_*` without defaults (F3, F8) | S | 0.2 |
| A2 | Neutral fixtures and examples; drop TeraPay/Housekeeper/`shopping-450413` (F5, F7) | S | A1 |
| A3 | One config approach: `.env.example`, `openpocket.config`, remove hardcoded owner/project/region (section 12) | M | A2 |
| A4 | Upstream guard: `ALLOWED_OWNERS`, refuse upstream writes, tests | M | A3 |
| A5 | Rename Viber → OpenPocket (code, env aliases keep `VIBER_*` working, `openpocket.yml`, PR title prefix, UI text) | M | A3 |
| A6 | Split CLAUDE.md (public / overlay); README rewrite incl. the Claude-terms notice (decision 22) | M | A5 |
| A7 | Schema: `users`, `provider_connections`, `usage`, `user_id` columns + backfill, `scope(userId)`, cross-user access tests | L | A5 |
| A8 | Token vault: envelope encryption, rotation, redaction patterns | M | A7 |
| A9 | MCP OAuth server (DCR, PKCE, refresh, metadata) and sign-in page | L | A7 |
| A10 | GitHub App: manifest/doc, sign-in, installation flow, HMAC + idempotent webhooks | L | A8, A9 |
| A10b | Spike: which GitHub permission lets the App **create repos for a personal account** (decision 15); create-from-starter flow | M | A10 |
| A11 | Account page: connections, disconnect/revoke, delete my account job, audit log | M | A8, A10 |
| A12 | Per-user notifications (ntfy/email), per-user task numbers, rate limits, **free-tier cap (default 100 tasks/month, adjustable)** | M | A7 |
| A13 | Engine interface; move today's Claude-Actions flow behind it | M | A7 |
| A14 | Spike then Copilot engine: issue, assignment, PR link, follow-up comments, stop; plain-words quota message with "Copilot Pro or Claude API key" | L | A10, A13 |
| A15 | Engine routing; **Claude API-key engine** (key written only as repo secret via write-only Secrets permission, never in the DB); hosted boot check refuses subscription tokens | M | A14 |
| A16 | Copilot checks on the **Vercel build** (no "Approve and run workflows" tap); optional Actions tests for users who switch them on | M | A14 |
| A17 | Spike: OpenPocket-managed Neon branches (decided) vs the integration fallback; check Vercel per-branch env API; use pasted tokens until partner approvals land | M | A8 |
| A18 | Connect Vercel: project from repo, env vars, **fixed three-environment setup** (5.6 option A: `production` branch protected for OpenPocket only, staging domain, env scopes), deployment status, Hobby non-commercial notice | L | A8, A17 |
| A19 | Connect Neon: project, `preview-seed` (fake data) and long-lived **`staging`** branches, branch per preview, cleanup on PR close | L | A17 |
| A20 | Schema flow with **Atlas** (declarative `db/schema.sql`): plan in plain words, destructive flag, preview auto, staging on merge, prod only after the spoken code, never in the Vercel build (`schema_waits`) | L | A19 |
| A21 | Staging step for apps: "put on staging" tool and web button, status "On staging", staging link, "reset staging data on request" | M | A18 |
| A22 | Prod promote for apps: spoken-code gate, `production` fast-forward, Live detection, rollback hint | M | A21 |
| A22b | Enhanced-security toggle: ntfy default, **Twilio WhatsApp** code with approved template (hosted), own Twilio for self-hosters | M | A22 |
| A22c | **Custom domain** field in project settings: add to the Vercel production project (optional staging subdomain), show DNS records in plain words, recheck verification | M | A18 |
| A23 | Starter template `openpocket-starter`, stack rules files, build-version toast | M | A18 |
| A24 | Just-in-time prompts in MCP replies (connect Vercel/Neon, recommend database) | M | A18, A19 |
| A25 | Self-host mode: single tenant, setup/`doctor` (checks all three environments), PAT flow, token-based Vercel/Neon | L | A7, A18 |
| A26 | **Move OpenPocket's own server to Vercel + Neon** (decision 7): spike (Express on functions, MCP streaming, cron limits), then DB-backed jobs queue, webhook-driven refresh, "Deploy to Vercel" button. Expected: a handful of tasks | L | A25 |
| A27 | `docs/self-hosting.md` and dry run by a second person on fresh accounts | L | A26 |

### Stage B: private mirror and publishing reviewed changes to public

| Step | Task | Effort | After |
|---|---|---|---|
| B1 | Dani: create `openpocket-private` from the scrubbed tree; overlay; point its CI at Vercel/Neon (personal staging + production, as in 14.1) | M, Dani | A6, A26 |
| B2 | `scripts/publish.sh` (squashed neutral diff, **author Daniel Kupfer + GitHub no-reply**, excludes overlay), sync-down script, overlay CI check | M | B1 |
| B3 | Fresh public repo export `kupferco/openpocket`: LICENSE (MIT, Daniel Kupfer), initial commit, no history | S | B2, A27 |
| B4 | CONTRIBUTING (no CLA/DCO, best-effort support), SECURITY and CODE_OF_CONDUCT (alias contact), templates, CODEOWNERS. **No PRIVACY/TERMS yet** (decision 6) | M | B3 |
| B5 | Fork-safe CI (no secrets), secret scanning, branch protection | M | B3 |
| B6 | Dani: trademark check passed; tidy old `kupferco` repos (private/archived) **closer to launch**; make the public repo public; announce nothing | S, Dani | B4, B5 |
| B7 | First reviewed change published through `publish.sh` as a trial run, merged back down | S | B6 |

### Stage C: Dani moves his own instance and projects from Viber to OpenPocket

| Step | Task | Effort | After |
|---|---|---|---|
| C1 | Freeze: no running tasks, no `prod_queued` | S, Dani | B7 |
| C2 | Deploy `personal-staging` from `openpocket-private` on Vercel + Neon, against a copy of Viber's database; run the additive migrations | M | C1 |
| C3 | Cut over: promote `personal-prod`, rotated secrets, new connector URL (15.1 steps 4-8) | L, Dani | C2 |
| C4 | Re-sync apps one at a time (workflow, CLAUDE.md block, hook secret): Pond/Housekeeper first, **TeraPay deferred until OpenPocket is running** (decision 19) | L | C3 |
| C5 | Week of running on OpenPocket; rollback path (Viber on Cloud Run) stays until D1 | S | C4 |

### Stage D: retire Viber

| Step | Task | Effort | After |
|---|---|---|---|
| D1 | Archive `kupferco/viber` as read-only history store (or make it private); remove its Cloud Run services; revoke old PAT/tokens and workload-identity binding | S, Dani | C5 |
| D2 | Remove `VIBER_*` aliases and the Cloud Run adapter after a quiet release | S | D1 |
| D3 | TeraPay pipelines onto OpenPocket (deferred earlier) | M | D1 |

### Stage E: hosted launch (later)

| Step | Task | Effort | After |
|---|---|---|---|
| E1 | **Legal first:** hosted operator/entity, `PRIVACY.md`, `TERMS.md`, GDPR (deletion, data location) with a **lawyer review** | M, Dani | D1 |
| E2 | Deploy `hosted-staging` and `hosted-prod` (separate from the personal ones); Dani's test account; Twilio number and template approved | M | E1, A22b |
| E3 | Landing page, connector instructions, closed beta with a few users (user-pasted Vercel/Neon tokens if partner approvals are not in yet) | M | E2 |
| E4 | Monitoring, abuse limits, runbooks, key-rotation drill, RLS second barrier | M | E3 |
| E5 | Open signup | S | E4 |
| E6 | Issue/PR triage routine, Dependabot | S | B6 |

Rough total: with hosting on Vercel as a handful of tasks, about 35-50 working days. Stage A is the bulk. The first usable slice for Dani is A1 → A13, A17-A22 and A26, then B and C; the hosted product also needs A14-A16 and Stage E.

---

## 17. Decisions from Dani and open questions

### 17.1 Decisions (Dani's answers, 13 Nov 2026)

| # | Topic | Decision | Applied in |
|---|---|---|---|
| 1 | Repo owner | `kupferco/openpocket`, no new org. Old `kupferco` repos tidied (private/archived) closer to launch. | 1, 13, B6 |
| 2 | Name | Keep "OpenPocket"; only the domain needs a suffix. No npm package in v1. Trademark check before launch. | 1, 0.1 |
| 3 | Commit identity | Real name Daniel Kupfer + GitHub no-reply email. | 11 (F4), B2 |
| 4 | MIT holder | Daniel Kupfer personally. | 13 |
| 5 | SECURITY / CoC contact | Dedicated alias (e.g. `security@` his domain) forwarding to Dani. | 13, 0.1 |
| 6 | Legal / privacy / ToS / GDPR | Parked; not needed for the open-source publish; required with lawyer review before hosted sign-ups. | 9, 13, E1 |
| 7 | OpenPocket's hosting | Vercel + Neon. Viber stays on Cloud Run until retired. A handful of Viber tasks, not 10-15 days. | 7, A26 |
| 8 | Engines | Claude API-key engine at launch (own Actions, no tap) plus Copilot. Copilot checks run on the Vercel build, no "Approve and run workflows" tap. Actions tests only if the user switches them on. | 4.5, 6 |
| 9 | Copilot quota | Docs: "works with your Copilot plan; free plan limited, Pro recommended". On quota, explain in plain words, suggest Copilot Pro or a Claude API key. | 3.1, 4.1a, 6.1 |
| 10 | Vercel Hobby | Non-commercial: tell users when they connect Vercel; don't enforce. | 3.1, 4.2 |
| 11 | Neon previews | OpenPocket-managed branches from a fake-data seed branch; never copy real data. | 5.3 |
| 12 | Claimable Neon | Not used at all for now; no throwaway demos in v1. | 5.5 |
| 13 | Prod schema changes | Declarative schema with Atlas; plan read in plain words; destructive flagged; applied to prod only after the spoken code; not in the Vercel build. | 5.3, A20 |
| 14 | Prod gate (hosted) | Spoken code AND branch protection so only OpenPocket can push to the production branch. Enhanced security is a toggle: hosted = WhatsApp via Twilio (approved template, small cost); self-hosters = ntfy or their own Twilio. | 4.6, 9, A22b |
| 15 | New repos | OpenPocket creates repos for users; needs an extra GitHub permission (verify which works for personal accounts). | 3.1, 4.1, A10b |
| 16 | Dani's instance | Migrate his projects to his own self-hosted OpenPocket and retire Viber, before the hosted launch. | 15, C, D |
| 17 | Dani's platform | Vercel + Neon too. | 7, 15 |
| 18 | Order | Open source first, hosted later. `docs/saas-plan.md` merged here (Appendix A), superseded (step 0.3, done). | 0, 16, App. A |
| 19 | TeraPay | Deferred until after OpenPocket is running. | C4, D3 |
| 20 | Support / quota | "As is, best effort" via GitHub issues, no response-time promise. Hosted free tier: generous cap (e.g. 100 tasks/month/user), adjustable. | 8, 10, 13, A12 |
| 21 | Contributions | No CLA, no DCO; inbound = outbound MIT. | 13 |
| 22 | Claude terms | README says subscription tokens are for personal use on your own instance only; no sharing, no service on them; hosted uses Copilot or API keys. | 0, 13 |
| 23 | API keys on hosted | Repo secret only, never in OpenPocket's DB (write-only Secrets permission). | 4.1, 4.5, 9 |
| 24 | Partner approvals | Apply now to Vercel and Neon; first beta uses user-pasted tokens until approved. | 4.2, 4.3, 0.1 |
| 25 | Environments | **Fixed**: preview (per task), staging (`main`, staging domain), production (`production` branch, promoted only with the spoken code); option A on every plan; no custom-environments variant; changing it means changing the source. | 2.1, 5.6 |
| 26 | Staging data | One long-lived Neon `staging` branch, fake data, reset only on request. | 5.6 |
| 27 | Two deployments | Dani's personal OpenPocket and the public hosted OpenPocket (he uses a test account), each with its own staging and production. | 2, 14.1 |
| new | Custom domain | Project settings have a custom domain field; OpenPocket adds it to the Vercel production project and says in plain words which DNS records to set (optional staging subdomain). | 3.1 (14d), 4.2, 8, A22c |

### 17.2 Open questions for Dani (new)

1. **GitHub permission for creating repos:** if no App permission works for personal accounts, may OpenPocket fall back to "create from template" by the user's own tap?
2. **Domain:** which name and suffix for the hosted OpenPocket and the staging subdomain of OpenPocket itself?
3. **WhatsApp template:** is a Twilio number and Meta template approval acceptable as a hosted prerequisite, with ntfy as the fallback if it is delayed?
4. **Atlas:** OK to depend on Atlas (Apache-2.0 core; check which features need the paid cloud) for the schema plans, or should OpenPocket write the diff itself?
5. **Branch protection on a Hobby/free repo:** it may be limited on private repos of free accounts; if blocked, should OpenPocket refuse hosted production deploys or fall back to the spoken code alone?
6. **Old `kupferco` repos:** which ones should be private or archived, and when (before B6 or after the hosted launch)?
7. **Free-tier cap:** is 100 tasks a month per user the right start, and should heavy users be asked to add a Claude API key?

---

## Appendix A: merged from `docs/saas-plan.md` (superseded)

The earlier `docs/saas-plan.md` (from the private Viber repo) is superseded by this plan; nothing in it is lost. What it still contributes:

- **Claude access rules.** The published Claude Code terms ([legal and compliance](https://code.claude.com/docs/en/legal-and-compliance)) do not allow a third-party service to offer Claude.ai login or route requests through Free/Pro/Max/Ultra credentials for its users, or to collect or store Claude.ai credentials. A user's own API key, billed to them, is allowed. Hence: hosted = API key or Copilot only; subscription tokens only on a self-hosted instance for the owner's personal use. Not confirmed in writing by Anthropic; whether Dani's own personal instance is comfortable "at scale" is worth one email to Anthropic. (The Register and Gigazine coverage in Feb 2026 was consistent with the official page; later billing-change reports were not confirmed.)
- **API key handling.** Validate once with a cheap call, show only the last four characters, never log (redaction patterns for `sk-ant-`), recommend a dedicated key with a monthly spend limit in the Console. Here the key lives only as a repo secret (decision 23).
- **Per-user limits (starting points).** Projects 5, concurrent runs 2, tasks started per day 30, backlog 50, prod deploys per hour 6, MCP requests per minute 60, progress events 500 per task kept 14 days. Adjustable by the admin; the monthly task cap is decision 20.
- **Pricing.** Free to start. If charging later: free tier plus a flat paid plan (or per seat for teams); usage-based pricing was judged poor because users already pay Claude per use. Rough marginal cost to Dani per active user $0.20-1 per month plus a fixed base (section 10 has the current numbers).
- **User guide for getting an Anthropic API key:** Console sign-up → billing and a spend limit → API keys, create a key named "OpenPocket" → paste it once; delete it in the Console to stop use. Verify menu names before publishing.
