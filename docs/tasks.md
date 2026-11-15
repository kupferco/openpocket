# OpenPocket: implementation task breakdown

Source: `docs/plan.md` (decisions in section 0 and 17.1, build order in section 16, open questions in 17.2). This file turns the plan into **57 Viber tasks** (T1 to T57). Nothing here reopens a decision: the stack (Node, Vite, Vercel functions, Neon), the fixed three environments (preview, staging, production), the engines (Copilot and Claude API key on hosted, subscription token only on self-host), Atlas, Twilio WhatsApp and the two separate deployments all stay as decided.

## Overview

The work follows the plan's order: **build in private, publish open source, move Dani's instance and retire Viber, launch hosted later.** Stage A (T1 to T41) scrubs and renames the current Viber code, makes it multi-user, adds the GitHub App, MCP OAuth, the Copilot and Claude API-key engines, the Vercel and Neon integrations with the fixed three environments, Atlas schema changes, staging and production promotion, self-host mode and the port of OpenPocket's own server to Vercel + Neon. Stage B (T42 to T47) sets up the private mirror and the public repo. Stage C and D (T48 to T51) migrate Dani's instance and retire Viber. Stage E (T52 to T57) covers the hosted launch. Pure Dani actions (accounts, registrations, legal, flipping switches) are listed in "Dani-only steps" and inside each task under "Dani: manual steps".

**Total: 57 tasks** (about 20 S/M, 30 M/L, a handful of spikes). The first usable slice for Dani is T1 to T8, T17 to T19, T24 to T32 and T38 to T41, then Stage B and C. The hosted product also needs T20 to T23, T33, T52 onward.

## Conventions for Dani creating the tasks

- **Where the tasks run.** Per plan section 16, Stage A and the migration tasks change the code that today lives in the private `kupferco/viber` repo; from T42 on they run on `openpocket-private`. Dani: before creating T1, copy `docs/plan.md` and this file into the target repo (same paths) so each brief's "read the plan" instruction works. If the Viber project points at `kupferco/openpocket` (docs only for now), the code must be imported there first; that is the one thing this file cannot decide for you.
- **Dependencies.** `after_live` (default) means the dependency is merged and deployed to production in Viber. `after_ready` means only its preview must be ready (used for spikes, docs and work that only needs the code to branch from). Independent tasks have no dependency and can run in parallel; where two parallel tasks touch the same files, the later one rebases.
- **Every brief** starts with the shared preamble below; paste it before the task's own text. Each task is sized for one reviewable sitting.
- **Open questions.** A task marked "Open question" names the 17.2 question and the assumed default it will build with; changing the answer later is a small follow-up, not a blocker.
- One deliberate deviation from plan order: the durable jobs table (plan section 7, originally part of A26) is built early as T8, because account deletion (T16), prod batches and cleanup need it.

### Shared preamble (paste at the top of every brief)

> You are working on OpenPocket (formerly Viber). Read `docs/plan.md` first, in particular sections 0, 2.1, 8 and 9 and the sections named in this brief. Decisions in section 17.1 are final; do not reopen them. Rules: Node LTS, TypeScript, npm; schema changes are additive and `IF NOT EXISTS`; never put secrets, tokens, personal data or real project names in code, tests, logs or commits; hosted mode never accepts Claude subscription tokens; production only moves with the spoken code; previews and staging never touch production data; every query for user-owned data is scoped by user once T7 exists. Write tests for what you add. Run typecheck, tests and build before the last commit and fix what they find. Do not deploy. Put anything Dani must do by hand under "Dani: manual steps" in the pull request note, and use the `::viber-manual::` marker if a step cannot continue without him.

## Dependency graph

Arrow means "must wait for". Tasks on the same line after `||` can run in parallel.

```
Stage A1  scrub/rename/config
T1 -> T2 -> T3 (guard)  ||  T4 (rename) -> T5 (docs/README)
                              T4 -> T6 -> T7 || T8 || T9
                              T4 -> T13 (starter template)

Stage A2  identity and GitHub
T7 -> T10 (MCP OAuth);  T9 + T10 -> T11 (GitHub App sign-in) -> T12 (webhooks, needs T8)
T11 + T13 -> T14 (repo creation)
T9 + T11 -> T15 (account page) -> T16 (delete account, needs T8)
T7 -> T17 (notifications, task numbers) || T18 (limits, free tier) || T19 (engine interface)

Stage A3  engines
T11 + T19 -> T20 (Copilot assign) -> T21 (Copilot lifecycle, needs T12) -> T22 (routing + API-key engine)
T21 + T13 -> T23 (Copilot checks on Vercel build)

Stage A4  Vercel, Neon, environments
T9 -> T24 (spike) -> T25 (Vercel) -> T26 (three environments) -> T30 (staging) -> T31 (promote)
T25 -> T27 (Neon) -> T28 (branch per preview, needs T12) || T29 (Atlas plan)
T29 + T31 -> T32 (prod schema approval)
T31 -> T33 (enhanced security);  T26 -> T34 (custom domain);  T25 + T27 -> T35 (just-in-time prompts)

Stage A5  self-host and platform
T7 + T26 -> T36 (single tenant) -> T37 (setup/doctor) ;  T36 + T8 -> T38 (spike) -> T39 (port) -> T40 (deploy button)
T37 + T40 -> T41 (self-hosting guide)

Stage B   T5 + T39 -> T42 (private mirror) -> T43 (publish.sh) -> T44 (public export, also needs T41)
          T44 -> T45 (community files) || T46 (fork-safe CI) -> T47 (go-public checklist, needs both)
Stage C/D T39 + T6 -> T48 (migration script + runbook) ;  T4 -> T49 (app re-sync) ;  T48 + T49 -> T50 (remove aliases) ;  T49 -> T51 (TeraPay)
Stage E   T22 + T33 + T18 + T39 -> T52 (hosted deployment) ; T25 + T27 -> T53 (Vercel/Neon OAuth) ;
          T52 -> T54 (landing) -> T55 (monitoring, runbooks) ;  T52 + T7 -> T56 (RLS) ;  T47 -> T57 (triage, Dependabot)
```

**Parallel groups** (once their listed dependencies are live): {T3, T4} after T2; {T7, T8, T9, T13} after T6/T4; {T17, T18, T19} after T7; {T20.., T24} start independently once T11 and T9 exist; {T28, T29} after T27; {T33, T34} after T31/T26; {T45, T46} after T44; {T53, T56, T57} are free-standing late tasks. The long critical chain is T1, T2, T4, T6, T9, T24, T25, T26, T30, T31, T36, T38, T39, T42 to T44.

## Open questions from section 17.2 and the tasks they affect

| # | Question (plan 17.2) | Assumed default | Tasks |
|---|---|---|---|
| Q1 | If no App permission can create repos for personal accounts, may OpenPocket fall back to "create from template" by the user's own tap? | Yes, fall back to a one-tap "Use this template" link. | T14 |
| Q2 | Domain: which name and suffix for hosted OpenPocket and the staging subdomain? | Configured by `PUBLIC_URL` and `STAGING_URL` env vars; no domain hardcoded; placeholder `openpocket.example`. | T34, T52, T54 |
| Q3 | Twilio number plus Meta template approval as a hosted prerequisite, ntfy as fallback if delayed? | Yes: ntfy is the default and the fallback; WhatsApp is enabled only when Twilio config is present. | T33, T52 |
| Q4 | Depend on Atlas (Apache-2.0 core) for schema plans, or write the diff ourselves? | Depend on Atlas open-source core only; the task checks no paid-cloud feature is needed. | T29, T32 |
| Q5 | Branch protection unavailable (free private repos): refuse hosted production deploys, or fall back to the spoken code alone? | Fall back to the spoken code alone, with a plain-words warning and a `doctor` warning; do not block. | T26, T31, T37 |
| Q6 | Which old `kupferco` repos go private or archived, and when? | Not part of any coding task; T47 prepares the checklist with a placeholder list; Dani decides. | T47, T51 |
| Q7 | Is 100 tasks a month the right free cap; should heavy users be asked to add a Claude API key? | 100 per month, adjustable in `instance_settings`; at 80% show an optional "add a Claude API key" hint. | T18 |

## Dani-only steps (no Viber task)

Do these at the point named; the tasks that need them say so under "Dani: manual steps".

| When | Step (plan ref) |
|---|---|
| Now (0.1) | Apply for the Vercel integration listing and the Neon OAuth/partner app (decision 24); trademark check for "OpenPocket"; buy the domain (Q2); create the `security@` alias forwarding to you; set git identity Daniel Kupfer + GitHub no-reply email. |
| Before T1 (0.2) | Run gitleaks/trufflehog on tree and history, review `docs/screenshots/*.png`, rotate the section 11 secrets. |
| After T47 (B6, B7) | Pass the trademark check, tidy old `kupferco` repos (Q6), make `kupferco/openpocket` public, run the first trial publish through `publish.sh` and merge it back down. |
| C1, C3, C5 | Freeze Viber tasks, run the cut-over from the T48 runbook, run a week on OpenPocket. |
| D1 | Archive `kupferco/viber` (or make it private), remove its Cloud Run services and workload-identity binding, revoke old tokens. Create T50 and T51 only after this. |
| E1 | Hosted legal: entity, `PRIVACY.md`, `TERMS.md`, GDPR, lawyer review. Required before T52 goes to production and before open signup. |
| E5 | Open signup (flip the flag from T52) after T55 is live. |

---

# Stage A: build OpenPocket in the private repo

## T1: Scrub personal data and fixtures
- **depends_on:** none (Dani: do step 0.2 first)
- **Size:** S | **Plan:** section 11 (F3, F5, F7, F8), section 12, steps A1 and A2
- **Brief:** Remove personal and project-specific data from the tree. Replace the owner's email and `noreply@` sender defaults (`.env.example`, `config.ts`, `scripts/deploy.sh`) with empty/required values; email notifications are off unless `NOTIFY_EMAIL` and `NOTIFY_FROM` are set. Replace the personal name in comments and user-facing strings with "the owner" and add an `OWNER_NAME` setting (default "the owner") used by those strings and the CLAUDE.md block written into apps. Replace real fixtures (the other GCP project id, deployer service account, evidence bucket) with `example-project-123` and `myapp-deployer@example-project-123.iam.gserviceaccount.com`. Replace real project names (Housekeeper, TeraPay, the lv-notas repo, Pond) in tests, UI placeholders, README and MCP descriptions with neutral ones (`acme/webapp`, "Webshop"); drop the TeraPay README line and the Housekeeper legacy-marker branch in `onboard.ts`. Add a CI-runnable test or script that greps the tree for the removed terms and fails if they come back.
- **Done when:** a repo-wide search for the personal email, personal name, other-project ids and real project names finds nothing outside git history; typecheck, tests and build pass; behaviour is otherwise unchanged.
- **Dani: manual steps:** none (do the 0.2 scan and secret rotation yourself first).

## T2: One config approach
- **depends_on:** T1 (after_live)
- **Size:** M | **Plan:** section 12 (proposal and inventory), step A3
- **Brief:** Introduce one place per kind of setting: env vars validated in `config.ts` (document each in `.env.example`) and a committed non-secret `openpocket.config.json` (owner, repo, notify sender, owner name, service/project prefixes). Remove hardcoded owner repo, GCP project, region and service names from code, workflows and scripts by reading them from config or repo variables (`vars.*`); default the upstream repo from `GITHUB_REPOSITORY`. Make `onboard.ts` write `uses: <OPENPOCKET_REPO>/.github/actions/report@main` (the instance's own repo) instead of a hardcoded one. Read per-tenant values through `config.ts` accessors, not module constants. Add `npm run setup` that writes both files interactively and sets the GitHub repo variables with `gh`.
- **Done when:** no hardcoded owner, project, region or repo name remains in code or workflows (tests prove the report-action reference follows the configured repo); `.env.example` lists every variable; existing deployments keep working with their current env.
- **Dani: manual steps:** none.

## T3: Upstream write guard
- **depends_on:** T2 (after_live)
- **Size:** S | **Plan:** section 12 "Guard", decision 4, step A4
- **Brief:** Add a hard allow-list: every GitHub write (branches, PRs, comments, merges, secrets, workflow files) is refused if the repo owner is not the token's user or in `ALLOWED_OWNERS`, and refused for the upstream public repo unless `ALLOW_UPSTREAM=true`. Implement it once in the GitHub client wrapper so no call path bypasses it.
- **Done when:** tests show a task on a fork never targets the upstream remote, writes to a non-allowed owner fail with a plain-words error, and the allow-list covers every existing write call.
- **Dani: manual steps:** none.

## T4: Rename Viber to OpenPocket
- **depends_on:** T2 (after_live)
- **Size:** M | **Plan:** section 12 "Additional hardcoded items", step A5
- **Brief:** Rename in code, UI, docs, workflows and scripts: `viber.yml` to `openpocket.yml`, `VIBER_*` env vars to `OPENPOCKET_*`, `viber-*` service/secret/branch names, the PR title prefix `Viber #n` to `OpenPocket #n`, `HOOK_RULES` text. Keep the old `VIBER_*` names working as aliases (one release, with a log warning) and keep existing `viber.yml` readable so Dani's instance and apps keep working. Do not touch Cloud Run service names in the live deploy workflow beyond what config (T2) allows.
- **Done when:** grep for "viber" finds only the alias layer, its tests and history notes; an instance configured with only old `VIBER_*` vars still boots and passes its tests; new names work.
- **Dani: manual steps:** none.

## T5: Split CLAUDE.md and rewrite README
- **depends_on:** T4 (after_live)
- **Size:** M | **Plan:** section 13 (CLAUDE.md, README rows), decision 22, step A6
- **Brief:** Split `CLAUDE.md` into a public generic file (shape, commands, generic decisions) and a `overlay/CLAUDE.private.md` template for personal rules (kept out of public by the T46 check). Rewrite `README.md`: what OpenPocket is, the voice-first loop, the screenshots placeholder, a plain Claude terms notice (subscription tokens are for personal use on your own instance only; sharing them or running a service on them is not allowed; hosted uses Copilot or API keys), the prod safety model (two-call spoken code, optional enhanced security, DB-change approval, branch/PR-only coding runs), a quick start link to `docs/self-hosting.md` (written in T41), and the "this runs your code with your credentials" warning.
- **Done when:** README and public CLAUDE.md contain no personal names or private deployment details; the Claude notice is present; overlay template exists and is documented.
- **Dani: manual steps:** review the README wording.

## T6: Multi-user schema and backfill
- **depends_on:** T4 (after_live)
- **Size:** M | **Plan:** section 8 (tables, single-tenant rule), section 15.2, step A7 (part 1)
- **Brief:** Add additive tables `users`, `provider_connections`, `github_installations`, `usage`, `instance_settings`/`user_settings` split, `audit_log`, `webhook_deliveries`, and `user_id` (indexed) on `projects`, `tasks`, `task_rounds`, `progress_events`, `schema_waits`, `activity`, `codes`, plus the extra `projects` and `tasks` columns listed in section 8 (`engine`, `custom_domain`, Neon/Vercel ids, `production_branch` default `production`, `copilot_issue`, `neon_branch`). Per-user task numbers via `user_task_seq` (internal id stays global). Backfill: create `users.id = 1` for the owner and set `user_id` on every existing row; `OPENPOCKET_MODE=single` keeps everything working with the implicit user. Include `is_test` on users.
- **Done when:** migrations run twice without error on an empty DB and on a copy of a populated one; existing tests pass unchanged in single mode; a test shows every old row gets `user_id = 1`.
- **Dani: manual steps:** none.

## T7: User scoping helper and isolation tests
- **depends_on:** T6 (after_live)
- **Size:** L | **Plan:** section 8 "Rules", section 9 "Tenant isolation", step A7 (part 2)
- **Brief:** Add a `scope(userId)` data-access layer and move every route handler and MCP tool onto it; no raw table access in handlers. Make the prod queue per (user, project). Write tests: user A cannot read, start, merge, deploy or see user B's task or project, for each MCP tool and HTTP route (table-driven so new routes must be added to it).
- **Done when:** a lint or test fails if a handler touches tables directly; the isolation test matrix passes for every existing tool and route; single mode behaves as before.
- **Dani: manual steps:** none.

## T8: Durable jobs table and runner
- **depends_on:** T6 (after_live)
- **Size:** M | **Plan:** section 7 (re-platforming bullets), section 8 (`jobs`)
- **Brief:** Add a `jobs` table (kind, payload, run_at, locked_until, attempts, last_error) with lease-locking, backoff and idempotent handlers, and a runner invoked by a cron-style HTTP endpoint (protected by a secret) and by pokes after webhooks. Move the in-memory prod-batch timers and the `refresh`/cleanup intervals onto jobs, behind the existing behaviour (the Node server keeps a local interval that calls the same runner until Viber is retired).
- **Done when:** tests cover double-claim, crash-and-retry, backoff and idempotency; the prod batch window works through jobs with no in-process timer; existing prod-queue tests pass.
- **Dani: manual steps:** none.

## T9: Token vault
- **depends_on:** T6 (after_live)
- **Size:** M | **Plan:** section 9 (tokens at rest, revocation), step A8
- **Brief:** Envelope encryption with AES-256-GCM: a key-encryption key from `ENCRYPTION_KEY` wraps a per-record data key; ciphertext carries a key-version prefix; implement key rotation (re-wrap) as a job. Store provider tokens in `provider_connections` through it only. Extend `redact.ts` with GitHub App (`ghu_`, `ghs_`, `ghr_`), Vercel and Neon token patterns. No API or MCP tool may return a token.
- **Done when:** round-trip, tamper-detection, rotation and redaction tests pass; a test scans API/MCP responses for token fields.
- **Dani: manual steps:** none (a KEK is generated per deployment later).

## T10: MCP OAuth authorization server
- **depends_on:** T7 (after_live)
- **Size:** L | **Plan:** section 4.4, section 9 (sessions/MCP tokens), step A9
- **Brief:** Implement OAuth 2.1 for the MCP connector: protected-resource and authorization-server metadata, dynamic client registration, PKCE, refresh tokens with rotation and reuse detection, access tokens 1 hour, all stored SHA-256 hashed (`oauth_clients`, `oauth_codes`, `oauth_tokens`). Allow-list Claude's callback `https://claude.ai/api/mcp/auth_callback` (verify against current Claude connector docs) with strict redirect-URI matching. Build the `/authorize` page and a pluggable "identity provider" hook; for now sign-in works with the single-mode admin password and a stub provider used by tests (GitHub arrives in T11). Bind MCP tokens to a `user_id` and make the MCP endpoint accept either the old path secret (single mode) or a bearer token. Session cookie `HttpOnly; Secure; SameSite=Lax` plus CSRF protection on state-changing routes.
- **Done when:** a full code+PKCE+refresh flow passes in tests, reuse of a refresh token revokes the family, and the old path-secret MCP URL still works in single mode.
- **Dani: manual steps:** none yet (the connector is added in Claude in T52).

## T11: GitHub App sign-in and installation
- **depends_on:** T9, T10 (after_live)
- **Size:** L | **Plan:** sections 3.1 (steps 3-6), 4.1, step A10 (part 1)
- **Brief:** Add the GitHub App flow: "Sign in with GitHub" as the identity provider for T10 (parked authorize request, state + PKCE), callback that creates `users` and a `github` connection with encrypted user-to-server and rotating refresh tokens (refresh on expiry), installation setup URL (`/github/installed`) that stores `github_installations` and lists the repos, and the "choose a project" screen creating `projects` rows. Document the App registration exactly as in section 4.1 (permissions, events, callback and setup URLs, "request user authorization during installation", expire user tokens) in `docs/github-app.md`, plus an App manifest JSON for one-click registration. Write the `.openpocket/` marker and rules files by PR (not direct push), reading the stack rules from section 5.1.
- **Done when:** flow works end to end against a mocked GitHub in tests; docs list every permission and event; tokens never appear in logs.
- **Dani: manual steps:** register the GitHub App from the manifest (one for personal-staging, later one per deployment: personal-prod, hosted-staging, hosted-prod, per decision 27); set `GITHUB_APP_ID`, `GITHUB_APP_PRIVATE_KEY`, `GITHUB_APP_CLIENT_ID/SECRET`, `GITHUB_WEBHOOK_SECRET` as secrets; install it on a test repo.

## T12: GitHub webhook receiver
- **depends_on:** T11, T8 (after_live)
- **Size:** M | **Plan:** section 4.1 (events), section 9 (webhooks), section 7 (webhooks replace polling)
- **Brief:** Add `POST /hooks/github` with HMAC verification, replay protection through `webhook_deliveries.delivery_id`, resolution to the user via `installation_id`, and handlers for `installation`, `installation_repositories`, `pull_request`, `issue_comment`, `check_suite`, `workflow_run`, `github_app_authorization`. Update task and PR state from events (replacing the polling sweep for hosted; keep the sweep in single mode as the fallback), mark connections `revoked` on `github_app_authorization` and `installation deleted` and notify "Reconnect GitHub". Poke the T8 runner after relevant events.
- **Done when:** tests cover bad signature, duplicate delivery, each event type and revocation; a PR event updates the right user's task only.
- **Dani: manual steps:** none beyond T11's webhook secret.

## T13: Starter template and stack rules
- **depends_on:** T4 (after_live)
- **Size:** M | **Plan:** section 5.1, section 5.4, step A23
- **Brief:** Create `templates/starter/` (the future `openpocket-starter` repo): Vite + React, `api/health.ts`, a test, `vercel.json`, `db/schema.sql` placeholder, a tiny `build-version.json` generated at build and a client snippet that polls it every 10 s and shows a "New version ready: refresh" toast (auto-refresh in preview only). Write the rules files (`copilot-instructions.md`, `AGENTS.md`, `CLAUDE.md`) as templates: use the stack, never add Next.js, edit only `db/schema.sql` for schema, no secrets in code, do not deploy, mark `needs_database` in PR descriptions. Update onboarding to write these rules to existing repos by PR and to ask before adding `vercel.json`/`api/` to an existing repo.
- **Done when:** the starter installs, tests, builds and typechecks in CI (add a job); the rules templates are rendered by onboarding in a test.
- **Dani: manual steps:** after merge, publish `templates/starter/` as the GitHub template repo `kupferco/openpocket-starter` (mark "Template repository").

## T14: Create repos for users (spike and flow)
- **depends_on:** T11, T13 (after_live)
- **Size:** M | **Plan:** section 3.1 step 6, section 4.1 (Administration row), decision 15, step A10b
- **Brief:** Spike which GitHub permission lets the App create a repo for a personal account from the starter template (`POST /repos/{template}/generate` with the user-to-server token; test "Administration: write" and alternatives); write findings to `docs/spikes/repo-creation.md`. Build "Create a new repo for me" in the project chooser using what works. If no App permission works for personal accounts, fall back to a one-tap "Use this template" link that sends the user to GitHub's template page and then to the installation chooser.
- **Done when:** the spike doc states which permission works and what was tested; the flow works in tests with mocked GitHub for both paths.
- **Open question:** Q1. Assumed default: yes, fall back to the one-tap template link.
- **Dani: manual steps:** create a scratch personal GitHub account (or use a test account) to verify the permission live and paste the result into the spike doc.

## T15: Account page, revocation and audit log
- **depends_on:** T9, T11 (after_live)
- **Size:** M | **Plan:** section 3.1 step 15, section 9 (revocation), section 8 (`audit_log`), step A11 (part 1)
- **Brief:** Build the web Account page: connected accounts (GitHub now; Vercel and Neon later) with **Disconnect** (calls the provider's revoke endpoint where one exists, deletes stored tokens, marks the connection `revoked`, pauses tasks that need it), engine and API-key status (set/removed, never the value), and a connections list driven by data so T25/T27 only add providers. Add `audit_log` writes (connect/disconnect/deploy/delete/token_use; no secrets) and an admin kill switch "revoke all tokens of a user".
- **Done when:** disconnect and kill-switch tests pass, paused tasks resume on reconnect, audit rows never contain token values.
- **Dani: manual steps:** none.

## T16: Delete my account
- **depends_on:** T15, T8 (after_live)
- **Size:** M | **Plan:** section 9 (deleting a user), step A11 (part 2)
- **Brief:** "Delete my account" on the Account page (MCP only asks the user to confirm in the web UI). A job, idempotent and resumable, does: revoke and delete provider tokens, stop and pause running tasks, delete the user's rows (`users`, `provider_connections`, `tasks`, `task_rounds`, `progress_events`, `oauth_*`, `notify_channels`, `codes`, `usage`) cascading by `user_id`, keep a tombstone (hash of `github_id`, `deleted_at`), send a final confirmation. Do not touch the user's repos, Vercel projects or Neon databases; list them with links on the confirmation page.
- **Done when:** a test deletes a populated user and proves no rows remain except the tombstone, other users are untouched, and a crash midway resumes cleanly.
- **Dani: manual steps:** none.

## T17: Per-user notifications and task numbers
- **depends_on:** T7 (after_live)
- **Size:** M | **Plan:** section 8 (`notify_channels`, task numbers), section 12 (single ntfy topic row), step A12 (part 1)
- **Brief:** Add `notify_channels` per user: generated secret ntfy topic, optional email (Resend, off unless configured), optional WhatsApp number (stored only; sending is T33). Replace the global `NTFY_TOPIC` for multi-user while single mode keeps it. Show the user's own numbers ("#1" per user) in MCP replies and the UI using `user_task_seq`. Add a UI to view, rotate and test the topic.
- **Done when:** two users get different topics and independent numbering in tests; single mode still uses `NTFY_TOPIC`.
- **Dani: manual steps:** none.

## T18: Rate limits, usage and free-tier cap
- **depends_on:** T7 (after_live)
- **Size:** M | **Plan:** section 8 (`usage`), section 9 (abuse), section 10, decision 20, Appendix A (per-user limits), step A12 (part 2)
- **Brief:** Add per-user limits (adjustable in `instance_settings`): projects 5, concurrent runs 2, tasks per day 30, backlog 50, prod deploys per hour 6, MCP requests per minute 60, open-task cap, brief length cap, progress events 500 per task kept 14 days. Add the monthly free cap (default 100 tasks per user) counted in `usage`; hitting any limit returns plain words and never silently drops work. At 80% of the monthly cap add an optional hint to add a Claude API key. Single mode has no cap.
- **Done when:** tests cover each limit, the monthly rollover, and the admin override; limits are config, not constants.
- **Open question:** Q7. Assumed default: 100 tasks/month, adjustable; hint at 80%.
- **Dani: manual steps:** none.

## T19: Engine interface
- **depends_on:** T7 (after_live)
- **Size:** M | **Plan:** section 2 (engines), section 6, step A13
- **Brief:** Define an `Engine` interface (start, continue, stop, status, link PR) with `tasks.engine` copied from `projects.engine` at start; move today's Claude Actions flow behind it as `claude_actions` with no behaviour change (workflow dispatch, progress hooks, `continue_task` on the same branch). Keep the subscription-token path available only when `ALLOW_SUBSCRIPTION_TOKEN=true`.
- **Done when:** all existing task tests pass through the interface; a fake engine exists for tests; engine selection reads the column.
- **Dani: manual steps:** none.

## T20: Copilot engine: spike and assignment
- **depends_on:** T11, T19 (after_live)
- **Size:** M | **Plan:** section 4.1a, section 6.1 steps 1-3, step A14 (part 1)
- **Brief:** First a short spike recorded in `docs/spikes/copilot.md` verifying against the current docs: how to assign an issue to `copilot-swe-agent` through GraphQL (`replaceActorsForAssignable`/`updateIssue`, `agentAssignment` fields, header `GraphQL-Features: issues_copilot_assignment_api_support`), that the user-to-server token is required, and whether a PR from the agent needs a manual "Approve and run workflows" click. Then implement `copilot` engine start: check the actor is assignable (`suggestedActors`), create the issue (title, brief, OpenPocket rules, label `openpocket`, hidden task-id comment), assign with `baseRef: main`, refresh the user token when needed.
- **Done when:** the spike doc exists with the verified input shapes; start works against a mocked GraphQL in tests; "Copilot not enabled" returns plain words with a link.
- **Dani: manual steps:** a GitHub test account with Copilot enabled and the staging GitHub App installed on a throwaway repo, to confirm the spike findings live.

## T21: Copilot engine: lifecycle and quota
- **depends_on:** T20, T12 (after_live)
- **Size:** L | **Plan:** section 6.1 steps 3-7 and the differences paragraph
- **Brief:** Link the Copilot draft PR to the task from the `pull_request` webhook (issue reference + author Copilot) and rename it `OpenPocket #n: title`. `continue_task` comments `@copilot <instructions>` on the PR with the user token. `stop_task` comments "stop", closes the PR, unassigns, and marks Paused even if the session cannot be cancelled. Detect a quota refusal and show "Your Copilot plan has run out for now. Upgrade to Copilot Pro, or add a Claude API key." and pause without silent retry. Progress for Copilot comes from the session log link and PR timeline (no live step feed). Before merge, sync the branch through the GitHub update-branch API.
- **Done when:** lifecycle tests (start, link, continue, stop, quota, sync) pass against mocks; branch names that are not `feat/…` work everywhere.
- **Dani: manual steps:** none.

## T22: Engine routing and Claude API-key engine
- **depends_on:** T21 (after_live)
- **Size:** M | **Plan:** sections 4.5, 6 (table and 6.3), decisions 8, 22, 23, 9 (subscription-token ban), step A15
- **Brief:** Implement routing rules 1-4 from section 6.3 and the engine choice screen (two cards, section 3.1 step 7: Claude API key or Copilot with detection). Build the **Claude API-key engine**: validate the key once with a cheap call, show only its last four characters, write it straight to the user's repo as the Actions secret `ANTHROPIC_API_KEY` through the App's write-only Secrets permission (it must never be stored in the DB, logs or memory beyond the request), write the `openpocket.yml` workflow by PR (the workflow never contains the key), and delete the secret when the project or key is removed. In hosted mode reject `sk-ant-oat…` tokens with HTTP 400 and refuse to boot when `OPENPOCKET_MODE=hosted` and `ALLOW_SUBSCRIPTION_TOKEN` is set. Add the user guide text for getting an API key and the spend-limit recommendation (Appendix A) to the engine card, flagged "verify menu names".
- **Done when:** tests prove the key never reaches the DB or logs, hosted + subscription token returns 400, the boot check refuses to start, and routing picks the right engine in each case.
- **Dani: manual steps:** none (menu names in the user guide get re-verified before launch).

## T23: Copilot checks on the Vercel build
- **depends_on:** T21, T13 (after_live)
- **Size:** M | **Plan:** sections 4.5, 6.1 step 5, decision 8, step A16
- **Brief:** For Copilot tasks run checks (typecheck, tests, build) as the Vercel **build command** so a preview deployment is itself the check and no "Approve and run workflows" tap exists. Add `build:check` to the starter and a `vercel.json` buildCommand written by onboarding. Show check state from the Vercel deployment on the task. Offer optional GitHub Actions tests as a switch; when on, show an "Approve checks" link for the Copilot PR (verify the "Require approval for Copilot" setting and document it).
- **Done when:** a failing typecheck fails the Vercel build in the starter and the task shows the failure in plain words; the optional Actions switch writes and removes the workflow cleanly.
- **Dani: manual steps:** none.

## T24: Spike: Neon branches and Vercel per-branch env
- **depends_on:** T9 (after_ready)
- **Size:** M | **Plan:** section 5.3, decision 11, step A17
- **Brief:** Verify with real API calls and write `docs/spikes/neon-vercel.md`: OpenPocket-managed Neon branches `preview/pr-<n>` from `preview-seed`, setting a per-git-branch `DATABASE_URL` through the Vercel env API (limits, reliability, cleanup), and the pasted-token path for both providers (Vercel token, Neon API key) used until partner approvals land. Include a recommendation; option A (the official Neon-Vercel integration) stays as the documented fallback. Add small client modules for the two APIs with tests against recorded fixtures.
- **Done when:** the doc has measured results and a go/no-go; the client modules and their tests exist.
- **Dani: manual steps:** create a Vercel account/team and a Neon account for testing, create a scoped Vercel token and a Neon API key, and store them as the repo secrets `SPIKE_VERCEL_TOKEN` and `SPIKE_NEON_API_KEY` (only used by this spike, delete after).

## T25: Connect Vercel
- **depends_on:** T9, T24 (after_live)
- **Size:** L | **Plan:** sections 3.1 step 10-11, 4.2, 5.6, decision 10, step A18 (part 1)
- **Brief:** "Connect Vercel" with a **pasted token** (OAuth arrives in T53): store it in the vault, add it to the Account page providers, create the Vercel project linked to the repo (production branch `production`), set env vars, read the deployment for a PR head SHA and post preview URL and state to the task, and notify "Preview ready" when the deployment for the new SHA is READY. Show the plain-words notice at connect time: "Vercel's free Hobby plan is for non-commercial use. If this app earns money you need Vercel Pro." (shown, not enforced). Mark the connection `revoked` on Vercel 401.
- **Done when:** with a mocked Vercel API, connect, project creation, deployment status and revocation work in tests; notice text is present; tokens never logged.
- **Dani: manual steps:** apply for the Vercel integration listing if not done (step 0.1); create a test token for manual verification.

## T26: Three environments (preview, staging, production)
- **depends_on:** T25 (after_live)
- **Size:** L | **Plan:** sections 2.1, 4.6, 5.6 (option A), decisions 14, 25, step A18 (part 2)
- **Brief:** Set up the **fixed** environment set on every project: production branch `production` (created from `main` if absent), staging = `main` as a Preview with a branch-assigned domain `<app>-staging.vercel.app` and branch-scoped env vars, previews per PR. No option for fewer or more. Apply GitHub branch protection so only the OpenPocket App can push to `production` (needs the Administration permission; the user is told in plain words when the plan does not allow it). Add a safety check that fails if a production `DATABASE_URL` is visible in Preview scope. Store `staging_subdomain`, `production_branch`, and per-environment state on `projects`.
- **Done when:** tests with mocks create all three, are idempotent, detect a leaked production env var, and handle "branch protection not available" with the fallback.
- **Open question:** Q5. Assumed default: fall back to the spoken code alone, with a plain-words warning; do not block hosted production deploys.
- **Dani: manual steps:** none.

## T27: Connect Neon
- **depends_on:** T24, T25 (after_live)
- **Size:** L | **Plan:** sections 5.2, 5.3, 4.3, decisions 11, 12, step A19 (part 1)
- **Brief:** "Connect Neon" with a **pasted API key** (OAuth in T53), stored in the vault, listed on the Account page. On "yes" to a database recommendation, create the Neon project `openpocket-<app>` with default branch `main` (production data), a `preview-seed` branch (schema from `db/schema.sql` plus fake seed data from a seed script) and one long-lived `staging` branch (fake data), and set `DATABASE_URL` per environment scope in Vercel: production to `main`, staging (scope for git branch `main`) to `staging`. Detect `needs_database` (the PR marker, or imports of `pg`/`@neondatabase/serverless`/`db/schema.sql`). Real data is never copied into `preview-seed`, previews or staging. Claimable Neon is not used.
- **Done when:** mocked-API tests cover project creation, the three branches, env var scoping, detection of the marker, and that no code path branches previews from `main`.
- **Dani: manual steps:** apply for the Neon OAuth/partner app if not done (step 0.1).

## T28: Neon branch per preview and cleanup
- **depends_on:** T27, T12 (after_live)
- **Size:** M | **Plan:** section 5.3 (branch per preview), step A19 (part 2)
- **Brief:** On PR open create branch `preview/pr-<n>` from `preview-seed` and set its `DATABASE_URL` in the preview scope for that git branch; on PR close (webhook) delete the branch and env var through a job with retries; record `tasks.neon_branch`. Add a reconciliation job that deletes orphans. Add the "reset staging data on request" primitive (recreate `staging` from the seed, request-only) used by T30.
- **Done when:** tests show the branch lifecycle, orphan cleanup and that production data never reaches a preview.
- **Dani: manual steps:** none.

## T29: Atlas schema planning
- **depends_on:** T27 (after_live)
- **Size:** L | **Plan:** section 5.3 (declarative schema with Atlas), decision 13, step A20 (part 1)
- **Brief:** Use Atlas (`atlas schema apply --dry-run`) from the OpenPocket job runner (not in the Vercel build) against the target Neon branch with `db/schema.sql` as the desired state. Spike first: how to run Atlas from the runner (binary or library), recorded in `docs/spikes/atlas.md`, including a check that only Apache-2.0 core features are used and which features need the paid cloud. Turn the plan into plain words ("adds a column `done` to `todos`; loses no data"), flag destructive changes (dropped tables/columns, narrowed types), compute a plan hash, and apply automatically to preview branches on creation and each push. Staging (apply on merge to `main`) is wired in T30.
- **Done when:** tests cover non-destructive, destructive and no-op plans, the plain-words output, the plan hash, and that applying a plan whose hash changed is refused.
- **Open question:** Q4. Assumed default: depend on Atlas open-source core only; if the spike finds a needed feature is paid, report it in the pull request and fall back to a minimal in-house diff for additive changes only.
- **Dani: manual steps:** none.

## T30: Staging for apps
- **depends_on:** T26 (after_live; also needs T28 and T29 only for apps with a database, which this task guards with a feature check)
- **Size:** M | **Plan:** sections 3.1 steps 14a-14b, 5.6, step A21
- **Brief:** Add the MCP tool `put_on_staging` and a web button: merge the PR into `main`, wait for the staging deployment (Vercel preview of `main` with the branch domain) to be READY, report "On staging" with the stable staging link on the task. For databases, apply the Atlas plan to the `staging` Neon branch on merge; require approval only if the plan is destructive. Add "reset staging data on request". Update status vocabulary (`on_staging`) in UI, MCP replies and notifications.
- **Done when:** tests cover merge, READY detection, destructive-needs-approval, non-destructive auto-apply and staging reset; no production resource is touched.
- **Dani: manual steps:** none.

## T31: Production promote for apps
- **depends_on:** T30 (after_live)
- **Size:** M | **Plan:** sections 2.1, 3.1 step 14c, 5.6 (`promote`), 9 (prod safety), step A22
- **Brief:** Rework `deploy_to_prod` for apps: the first call never promotes; it says what ships (checks, staging link, whether staging was tested) and returns the four-digit code bound to user, task, commit and hour. The second call with the code fast-forwards `production` to the commit proven on staging (never a force-push), then reports "Live" when the production deployment is READY and shows a rollback hint. Serialise production deploys per project, never deploy from a webhook alone, and work with branch protection on or off (see Q5).
- **Done when:** tests cover wrong code, expired code, different commit, double call, concurrent promotes, a non-fast-forward refusal and Live detection.
- **Open question:** Q5 (see T26); same assumed default.
- **Dani: manual steps:** none.

## T32: Production schema approval
- **depends_on:** T29, T31 (after_live)
- **Size:** M | **Plan:** section 5.3 table (production row), sections 5.6, 9, decision 13, step A20 (part 2)
- **Brief:** Before a promote that includes a schema change, apply the Atlas plan to the Neon `main` branch **only after its own spoken approval**: the plan in plain words, whether it loses data, and its hash are read out; the hash is re-checked immediately before applying; apply happens right before the production fast-forward through the job runner, never in the Vercel build. Use `schema_waits` to hold the promote until approved; abort leaves production untouched.
- **Done when:** tests prove no code path applies a plan to `main` without the approval, a changed plan after approval is refused, and an aborted wait changes nothing.
- **Open question:** Q4 (see T29).
- **Dani: manual steps:** none.

## T33: Enhanced-security code delivery
- **depends_on:** T31 (after_live)
- **Size:** M | **Plan:** section 4.6, section 3.1 step 14c, decision 14, step A22b
- **Brief:** Add a per-user **toggle** "enhanced security": the spoken code is not read out but sent to the user's channel. Default channel ntfy; on hosted, WhatsApp through a Twilio number with an approved message template; self-hosters can bring their own Twilio via env vars. Reuse `notify_channels` (T17). If Twilio config is missing or the template is not approved, fall back to ntfy with a plain-words notice. Redact numbers and codes in logs.
- **Done when:** tests with a mocked Twilio cover send, failure fallback to ntfy, and that the code is bound as before; the toggle is on the settings page.
- **Open question:** Q3. Assumed default: ntfy is the default and the fallback; WhatsApp only when Twilio config is present.
- **Dani: manual steps:** create a Twilio account and a WhatsApp sender number, submit the message template to Meta for approval, and set `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_WHATSAPP_FROM`, `TWILIO_TEMPLATE_SID` as secrets of the hosted deployments only (not personal).

## T34: Custom domain field
- **depends_on:** T26 (after_live)
- **Size:** M | **Plan:** section 3.1 step 14d, section 4.2 (Domains), section 8, step A22c
- **Brief:** Add a "Custom domain" field in project settings. On save, add the domain to the Vercel **production** project (and optionally `staging.<domain>` to the staging branch domain), read Vercel's required DNS records and verification state, and show the records in plain words ("Add this record at your domain provider: type CNAME, name `www`, value …"; A record for the bare domain). Recheck through a job until verified, then notify. Store `custom_domain`, `custom_domain_state`, `staging_subdomain`. OpenPocket never touches the user's DNS.
- **Done when:** mocked-Vercel tests cover pending, verified, error and removal; the plain-words instructions render for CNAME and A cases.
- **Open question:** Q2 (OpenPocket's own domain). This task only handles users' app domains and uses no OpenPocket domain; no default needed.
- **Dani: manual steps:** none.

## T35: Just-in-time prompts
- **depends_on:** T25, T27 (after_live)
- **Size:** M | **Plan:** section 3.1 steps 10 and 12, decision 7, step A24
- **Brief:** In MCP replies, prompt for a provider only when needed: connect Vercel at the first preview or publish ("To show you a preview I need your Vercel account. Open this link." with the Connect Vercel link), recommend a free Neon database when `needs_database` is detected ("Say yes to set it up"), and handle the yes flow including "Connect Neon" when absent. Never repeat the prompt after a decline within the task.
- **Done when:** tests walk first-task, first-preview, first-database journeys through the MCP tools with mocks and assert the exact prompts and that nothing is asked earlier than needed.
- **Dani: manual steps:** none.

## T36: Single-tenant mode
- **depends_on:** T7, T26 (after_live)
- **Size:** M | **Plan:** sections 3.2, 8 (last rules), step A25 (part 1)
- **Brief:** Make `OPENPOCKET_MODE=single` first-class: hide GitHub sign-in, use admin auth (`ADMIN_PASSWORD`, session cookie), run with the implicit user 1, keep the path-secret MCP URL (v1), a GitHub fine-grained **PAT** flow (setup page shows a pre-filled token-creation link and the exact permission list), and **pasted tokens** for Vercel and Neon (works without the GitHub App). Engines: `claude_actions` with `CLAUDE_OAUTH_TOKEN` or `ANTHROPIC_API_KEY`, or Copilot. Boot refuses hosted-only features in single mode and the reverse.
- **Done when:** the full test suite passes in single mode with no GitHub App configured; a single instance cannot be put into hosted behaviour through env alone.
- **Dani: manual steps:** none.

## T37: Setup page and `doctor`
- **depends_on:** T36 (after_live)
- **Size:** M | **Plan:** sections 3.2 steps 5 and "Environments for a self-hoster", step A25 (part 2)
- **Brief:** Build `/setup` and `npm run doctor` checks: token scopes, DB reachable, engine present, ntfy test, GitHub/Vercel/Neon tokens, and **all three environments** (creates and verifies preview, staging, production; none can be skipped), branch protection state (warn per Q5 default), and the "production `DATABASE_URL` visible in Preview scope" check. Output plain-words fixes. Subscription-token notice for single mode (personal use only).
- **Done when:** doctor passes against a healthy mocked setup and fails with a clear fix for each of the listed faults.
- **Open question:** Q5 (see T26); same assumed default.
- **Dani: manual steps:** none.

## T38: Spike: OpenPocket server on Vercel functions
- **depends_on:** T36, T8 (after_ready)
- **Size:** M | **Plan:** section 7, step A26 (spike)
- **Brief:** Verify in a throwaway Vercel project and write `docs/spikes/vercel-server.md`: Express (or Hono) exported as `api/index.ts`, MCP Streamable HTTP within max duration on Hobby and Pro, the cron limits (Hobby daily vs Pro per-minute), cold starts, and the Neon serverless driver. Recommend go or fall back to keeping Cloud Run as an "advanced" path (the rest of the plan is unaffected either way).
- **Done when:** the doc has measured limits and a clear go/no-go with the minimal code needed.
- **Dani: manual steps:** a Vercel project and Neon database for the spike, with `SPIKE_VERCEL_TOKEN` and `SPIKE_NEON_API_KEY` repo secrets (reuse T24's if still present).

## T39: Port OpenPocket's server to Vercel + Neon
- **depends_on:** T38 (after_live)
- **Size:** L | **Plan:** section 7 (bullets), decisions 7, 17, step A26
- **Brief:** Run the existing Express app as a Vercel function with one runtime-neutral handler; move cron-style work to Vercel Cron plus webhook pokes calling the T8 job runner (every minute on Pro, slower on Hobby, documented); secrets in Vercel env; Neon for the OpenPocket DB. Replace the Cloud Run preview pattern (`viber-preview-<n>` and `seed.ts`) with Vercel preview + a Neon branch per PR (dogfooding T25 to T28 flow) with `OPENPOCKET_ENV=preview`, no GitHub writes and no notifications, and the preview test login rules in `CLAUDE.md`. Keep the Node `server.ts` adapter until Viber is retired (T50). Add the `vercel.json` and a `deploy` workflow gated on repo variables so forks self-skip.
- **Done when:** the app builds and passes tests as a function locally (Vercel CLI dev), the job runner is driven by cron and webhooks, and nothing relies on in-memory state between requests.
- **Dani: manual steps:** create the Vercel projects `personal-staging`, `personal-prod` and the Neon project for the personal instance (per decision 27, separate from hosted), connect the repo, and set the env vars listed in `.env.example`.

## T40: Deploy to Vercel button
- **depends_on:** T39 (after_live)
- **Size:** S | **Plan:** section 3.2 step 2, decision 9
- **Brief:** Add the "Deploy to Vercel" button URL (clone URL for the user's fork, project name prompt, `env=` list: `ENCRYPTION_KEY` with a generator link, `ADMIN_PASSWORD`, `GITHUB_TOKEN`, `CLAUDE_OAUTH_TOKEN` or `ANTHROPIC_API_KEY`, `NTFY_TOPIC`) and a script that generates it; document and test (as far as possible without Vercel) whether the Neon Marketplace integration can be added in the same flow and injects `DATABASE_URL`; if not, document the second step. Add `docs/deploy-button.md`.
- **Done when:** the button URL is generated from config, documented, and the Neon-in-flow question has a recorded answer.
- **Dani: manual steps:** click the generated button once on a scratch Vercel account to confirm the env prompts and the Neon step.

## T41: Self-hosting guide
- **depends_on:** T37, T40 (after_live)
- **Size:** M | **Plan:** section 3.2, section 7, step A27
- **Brief:** Write `docs/self-hosting.md`: fork, deploy button, GitHub token, Claude token or API key or Copilot (with the personal-use-only notice), `/setup` and doctor, add the connector, add a repo, connect Vercel and Neon, the three fixed environments, cron limits on Hobby, the "never opens PRs upstream" guarantee, security notes (the instance holds a GitHub token), and troubleshooting. Add a short checklist for the second-person dry run.
- **Done when:** every step is traceable to a working feature and every command is copy-pastable; the checklist exists.
- **Dani: manual steps:** a second person follows the guide on fresh accounts and reports the stumbling blocks (a follow-up fix task if needed). Target: under 30 minutes, five manual actions.

---

# Stage B: private mirror and publishing

## T42: Private mirror overlay and CI
- **depends_on:** T5, T39 (after_live)
- **Size:** M | **Plan:** section 14, section 14.1, step B1
- **Brief:** Prepare what `openpocket-private` needs: a `scripts/export-tree.sh` that produces the scrubbed tree (`git archive`, no history) for the first commit, an `overlay/` directory (personal `CLAUDE.private.md`, config) with docs, and CI workflows for the private repo pointing at Vercel + Neon (preview per PR, `personal-staging` on `main`, `personal-prod` promoted by hand with the spoken code). Document the environments mapping from 14.1.
- **Done when:** running the export script yields a tree that passes gitleaks and the T1 term check; workflows are parameterised by repo variables.
- **Dani: manual steps:** create the private repo `kupferco/openpocket-private`, run the export script, make the initial commit with the Daniel Kupfer + GitHub no-reply identity, push it, add the overlay and secrets, point a Viber project at it.

## T43: publish.sh and sync-down
- **depends_on:** T42 (after_live)
- **Size:** M | **Plan:** section 14 (flow), decision 3, step B2
- **Brief:** Write `scripts/publish.sh <task>`: squashed diff of a reviewed task branch excluding `overlay/`, neutral commit message (no brief, no task number), author Daniel Kupfer with the GitHub no-reply email read from git config, a branch in the public repo and a draft PR opened by Dani's hand (the script never merges and never posts upstream automatically). Add `scripts/sync-down.sh` to fast-forward private `main` from public `main`, and a CI check on public that rejects `overlay/` paths.
- **Done when:** a test using two local bare repos shows the overlay excluded, the message neutral and the identity correct; sync-down handles a merged contributor PR.
- **Dani: manual steps:** set the git identity (Daniel Kupfer + no-reply) where the script runs.

## T44: Public repo export
- **depends_on:** T43, T41 (after_live)
- **Size:** S | **Plan:** section 13 (how to make the fresh repo), sections 1, 11 (manual checks), step B3
- **Brief:** Write `scripts/export-public.sh` and `docs/publish-runbook.md` for the one-time fresh public export: `git archive` of the scrubbed tree, new directory, `git init`, one initial commit by Daniel Kupfer (no-reply), push to `kupferco/openpocket`; never `filter-repo`. Include the section 11 manual checks (gitleaks and trufflehog on a clean clone made without credentials, screenshot review, search private Actions logs for pasted content). Verify `LICENSE` (MIT, `Copyright (c) 2026 Daniel Kupfer`) is present.
- **Done when:** the script produces a clean tree with no history in a temp dir and passes the term check and gitleaks; the runbook lists each manual check.
- **Dani: manual steps:** run the checks, review screenshots, run the script and push (the public repo stays private until T47's checklist passes). `kupferco/openpocket` already exists with docs only; decide whether to replace its history or push on top.

## T45: Community files
- **depends_on:** T44 (after_live)
- **Size:** M | **Plan:** section 13 (public repo setup), decisions 5, 20, 21, step B4
- **Brief:** Add `CONTRIBUTING.md` (issue first for large changes, PR rules, no CLA/DCO with inbound = outbound MIT, best-effort support via issues, Dani reviews every PR, the AI-generated-PR rules), `SECURITY.md` (GitHub private advisories, dedicated alias placeholder `security@<domain>`, threat notes), `CODE_OF_CONDUCT.md` (Contributor Covenant 2.1, same alias), issue templates (bug, feature, setup question to Discussions), a PR template with checklist (issue linked, tests, no secrets, AI disclosure), and `CODEOWNERS` (`* @kupferco`). No `PRIVACY.md`/`TERMS.md` yet (decision 6).
- **Done when:** all files exist with the alias as a clearly marked placeholder in one place.
- **Dani: manual steps:** create the `security@` alias and replace the placeholder; enable private vulnerability reporting and Discussions.

## T46: Fork-safe CI and repo hardening
- **depends_on:** T44 (after_live)
- **Size:** M | **Plan:** section 13 (CI for fork PRs, repo settings), section 9 (supply chain), step B5
- **Brief:** Workflows on `pull_request` (never `pull_request_target`) run typecheck, tests, build and gitleaks with no secrets and `permissions: contents: read`; deploy and preview jobs gate on variables and a same-repo head so forks self-skip; actions pinned by SHA; Dependabot config; add the overlay-path check. Add `scripts/harden-repo.sh` with the `gh api` calls for branch protection on `main` (require PR, check `check`, one review via CODEOWNERS, no force-push or deletion, push restricted to Dani), secret scanning with push protection, and "require approval for first-time contributors".
- **Done when:** workflows pass on a fork-like test (no secrets present) and the script is idempotent and has a `--dry-run`.
- **Dani: manual steps:** run `scripts/harden-repo.sh` on the public repo.

## T47: Go-public checklist
- **depends_on:** T45, T46 (after_live)
- **Size:** S | **Plan:** step B6, B7, section 11, section 1, decision 2
- **Brief:** Write `docs/launch-checklist.md`: trademark check passed, section 11 manual checks done, README and Claude notice reviewed, old `kupferco` repos tidied (list as placeholders; Dani decides which and when), repo visibility switched, nothing announced, then the trial run of `publish.sh` with one small reviewed change merged back down via `sync-down.sh`. Add a `scripts/preflight.sh` that automates the checkable parts (gitleaks, term check, presence of required files, no `overlay/`).
- **Done when:** the checklist and the preflight script exist and the script exits non-zero on a seeded violation in a test.
- **Open question:** Q6. Assumed default: placeholders; no repo is changed by a task.
- **Dani: manual steps:** the whole checklist (B6 and B7).

---

# Stage C and D: move Dani's instance and retire Viber

## T48: Viber-to-OpenPocket migration script and runbook
- **depends_on:** T39, T6 (after_live)
- **Size:** M | **Plan:** section 15 (all), steps C1 to C3, C5
- **Brief:** Write `scripts/migrate-from-viber.ts` that copies a Viber database into a **new** Neon database (read-only on the source), runs the additive migrations, creates user 1 and backfills, and verifies counts of projects, tasks and rounds. Write `docs/migration.md` as the exact runbook for Dani: freeze (no running tasks, nothing in `prod_queued`), deploy `personal-staging`, rotate secrets (the section 11 table: ntfy topic, `MCP_SECRET`, `HOOK_SECRET`, GitHub token, `CLAUDE_OAUTH_TOKEN`, `ENCRYPTION_KEY`, `ADMIN_PASSWORD`, DB credentials), promote `personal-prod` with the spoken code, switch the connector URL, run a week on it, rollback to Viber on Cloud Run until D1. The script must never write to the source DB.
- **Done when:** a test migrates a seeded populated DB and verifies counts and that source is unchanged; the runbook is step-numbered with rollback.
- **Dani: manual steps:** the runbook itself (C1, C3, C5).

## T49: App re-sync for migration
- **depends_on:** T4 (after_live)
- **Size:** M | **Plan:** section 15.1 step 7, step C4
- **Brief:** Make onboarding re-sync idempotent for apps already set up by Viber: replace `viber.yml` with `openpocket.yml` (remove the old file in the same PR), update the CLAUDE.md block, rotate the hook secret, and rewrite the report action reference to the instance's own repo. Add a `resync_project` MCP tool and web button operating one project at a time, and a dry-run that shows the diff in plain words.
- **Done when:** tests resync a fixture repo from a Viber-era layout, twice, with identical results the second time; the old workflow is gone and nothing else changed.
- **Dani: manual steps:** re-sync one app at a time after cut-over (a low-risk one first; the one that holds real users last, see T51).

## T50: Remove VIBER_* aliases and Cloud Run adapter
- **depends_on:** T48, T49 (after_live). **Dani: create this task only after step D1 is done.**
- **Size:** S | **Plan:** step D2, section 12, section 13 ("Dropped")
- **Brief:** Remove the `VIBER_*` env aliases, the `viber.yml` reading, `server.ts` Node adapter, `Dockerfile`, `scripts/deploy.sh` and the GCP parts of workflows and docs after a quiet release; delete the related config keys and tests.
- **Done when:** no alias or Cloud Run reference remains; the Vercel deployment is unaffected; tests pass.
- **Dani: manual steps:** D1 first (archive or make private `kupferco/viber`, remove Cloud Run services and workload-identity binding, revoke old tokens).

## T51: Move remaining pipelines to OpenPocket
- **depends_on:** T49 (after_live). **Dani: create this task only after step D1 is done.**
- **Size:** M | **Plan:** decision 19, step D3
- **Brief:** Bring the deferred app (the one with real users, held back by decision 19) onto OpenPocket: verify its older layout resyncs with T49's tool, check preview and staging behave as in 2.1 and that the preview test login follows the rules in `CLAUDE.md`, add a rollback note, and fix whatever the resync finds. Do not run anything against production data.
- **Done when:** a fixture of that app's layout resyncs cleanly in tests and the project's environment checklist passes in doctor.
- **Open question:** Q6 touches only the tidying of old repos; no default needed here.
- **Dani: manual steps:** run the resync and promote with the spoken code yourself.

---

# Stage E: hosted launch (later)

## T52: Hosted deployment configuration
- **depends_on:** T22, T33, T18, T39 (after_live). **Dani: E1 (legal) must be done before this reaches production.**
- **Size:** M | **Plan:** section 2 (two deployments), sections 9, 14.1, decision 27, step E2
- **Brief:** Make `OPENPOCKET_MODE=hosted` complete and separate from personal: its own Vercel projects (`hosted-staging`, `hosted-prod`), Neon database, GitHub App, OAuth apps and secrets via `.env.example` sections and `docs/hosted-deployment.md`. Add a **signup gate** (closed beta allow-list of GitHub logins, then an `OPEN_SIGNUP` flag), Dani's test account marked `is_test`, boot checks (no subscription tokens, KEK present, Twilio optional), and a Connect-to-Claude connector instructions page.
- **Done when:** hosted mode boots only with a valid config, signups outside the allow-list are refused with plain words, and `is_test` users are excluded from counts and metrics.
- **Open question:** Q2 (domain), Q3 (WhatsApp). Assumed defaults: domains come from `PUBLIC_URL`/`STAGING_URL`; ntfy is the fallback for codes.
- **Dani: manual steps:** pick and buy the domain (Q2), create the Vercel projects, Neon DB, a separate GitHub App and OAuth apps for hosted, set secrets, finish E1, and approve the Twilio template (T33).

## T53: Vercel and Neon OAuth connect
- **depends_on:** T25, T27 (after_live). **Create once the partner approvals land.**
- **Size:** M | **Plan:** sections 4.2, 4.3, decision 24
- **Brief:** Add the OAuth redirect flows ("Connect Vercel" integration and the Neon OAuth app with scopes `projects:create`, `projects:read`, `projects:modify`, `orgs:read`) next to the pasted-token paths, one consent each, just in time (T35), with refresh handling, revocation and smallest scopes. Keep pasted tokens as the fallback and for self-hosters. If Vercel's flow requires a Marketplace listing, document and implement the "Sign in with Vercel" alternative decided in the first pull request.
- **Done when:** mocked OAuth tests cover connect, refresh, revoke and the fallback to pasted tokens.
- **Dani: manual steps:** complete the Vercel integration listing and the Neon OAuth/partner approval; set the client ids and secrets for each deployment.

## T54: Landing page and connector instructions
- **depends_on:** T52 (after_live)
- **Size:** M | **Plan:** section 3.1 step 1, step E3
- **Brief:** Build the landing page at `PUBLIC_URL`: one sentence, an "Add to Claude" button showing the connector URL with a copy button, a 3-step illustration, plain facts (free to start, bring your own Vercel, Neon, Copilot or API key), and links to the repo and the Claude terms notice. Add the first-run checklist for closed-beta users (user-pasted Vercel and Neon tokens where OAuth is not yet approved).
- **Done when:** the page renders from config with no hardcoded domain and passes the UI check.
- **Open question:** Q2 (domain). Assumed default: uses `PUBLIC_URL`; placeholder text until the domain exists.
- **Dani: manual steps:** point the domain at the hosted production project; invite the beta users.

## T55: Monitoring, abuse limits and runbooks
- **depends_on:** T54 (after_live)
- **Size:** M | **Plan:** step E4, sections 9 (incident), 10
- **Brief:** Add health and uptime endpoints, structured logs without secrets, error alerts through ntfy to the operator, abuse dashboards (limits hit, failed sign-ins), and runbooks in `docs/runbooks/`: key-encryption-key rotation drill, "revoke all" switch, GitHub App key rotation, incident and `SECURITY.md` handling, backups and PITR. Add an automated rotation drill test.
- **Done when:** the drill passes in tests and each runbook has numbered steps and a verify line.
- **Dani: manual steps:** set up an external uptime check and run the key-rotation drill once on staging.

## T56: Row-level security second barrier
- **depends_on:** T52, T7 (after_live)
- **Size:** M | **Plan:** section 8 (RLS), section 9 (tenant isolation), step E4
- **Brief:** Add Postgres row-level security as a second layer on user-owned tables: `SET app.user_id` per request, policies per table, a non-owner application role, and migrations that are additive. Keep the `scope(userId)` helper. Extend T7's isolation matrix to run with RLS on and to prove a handler that forgets `scope` still cannot read another user's rows.
- **Done when:** the matrix passes with RLS on, and a deliberate raw query without scope returns no foreign rows in a test.
- **Dani: manual steps:** none (applied to hosted databases when deployed).

## T57: Issue triage routine and Dependabot
- **depends_on:** T47 (after_live)
- **Size:** S | **Plan:** step E6, decision 20
- **Brief:** Add labels, a triage workflow (label new issues, request missing version/log info for bug reports, close stale setup questions to Discussions), and a short `docs/maintaining.md` for the issue and PR routine (best effort, no promised response times, AI-PR policy). Confirm Dependabot is configured for npm and Actions.
- **Done when:** workflows run with `permissions` minimal and no secrets; the maintaining guide exists.
- **Dani: manual steps:** none.
