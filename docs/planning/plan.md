# skillassert.com: phased build plan

Status: approved by the owner 2026-10-29; execution Native, step by step. Implements `design.md` (approved 2026-10-29). Task-level plans live in
`plans/`, one per phase, written when the phase starts; `plans/phase-0-walking-skeleton.md` exists now.

## How to read this

- Phases are ordered by dependency and by the design's delivery guidance (section 16): skeleton first, content
  second, learner path third, accounts fourth, admin and pipeline fifth, operations and launch last.
- Sizes are hours of focused work for one developer learning the stack as they go, excluding reading. At 5 to 8
  hours a week, the week ranges follow. The six-week wish from round 1 buys the public trial quiz, not the whole
  learner path; accounts land around week nine to twelve.
- Every phase opens with a **Read first** list (official docs first, one deep dive per concept, at most three links
  per concept, one optional longer course). Links point at official documentation roots and well-known pages; if
  one has moved, search the same site before reading anything else.
- Every phase ends with an **Exit test** that a stranger could run. The phase is not done until it passes in CI and
  in production.
- Definition of done for every task inside a phase: failing test first, passing test, lint clean, Conventional
  Commit, pull request to `main`, CI green.

## Phase 0: Walking skeleton (12 to 16 h, weeks 1 to 3)

**Goal.** The thinnest slice through every layer, deployed: the Angular app on Cloudflare Workers at
skillassert.com reads the published category tree from the production Supabase project and renders it on a
prerendered home page; the NestJS API answers `GET /health` over HTTPS at api.skillassert.com after touching the
database; migrations, image and web build all flow from a merge to `main`.

**Deliverables.** Accounts and DNS (GitHub, Cloudflare zone with nameservers moved from Porkbun, Supabase project
in Frankfurt, VPS with SSH key); Nx workspace with `apps/web`, `apps/api`, `libs/shared/domain`, module-boundary
tags; local Supabase with the first migration (`category`, RLS, seed of the launch tree) and a pgTAP test; CI on pull
requests (lint, test, build, e2e against local Supabase); CD on `main` (migrations, API image to GHCR and Compose on
the VPS behind Caddy, web to Workers static assets with a 10-line fallback Worker); repo docs (GLOSSARY, ADRs,
design copied in).

**Exit test.** `curl -s https://skillassert.com/ | grep -c Angular` prints at least 1 from prerendered HTML;
`curl -s https://skillassert.com/quiz/start` returns 200 with the client shell; `curl -s
https://api.skillassert.com/health` returns `{"status":"ok","db":"ok"}`; a pull request shows green CI; `ss -ltnp`
on the VPS lists only ports 22, 80 and 443.

**Read first.**
- Nx: https://nx.dev/getting-started/intro and module boundaries https://nx.dev/features/enforce-module-boundaries
- Angular SSR and prerendering: https://angular.dev/guide/ssr ; hydration https://angular.dev/guide/hydration
- Supabase local development and migrations: https://supabase.com/docs/guides/local-development ;
  https://supabase.com/docs/guides/deployment/database-migrations
- Row-level security: https://www.postgresql.org/docs/current/ddl-rowsecurity.html and
  https://supabase.com/docs/guides/database/postgres/row-level-security
- NestJS: https://docs.nestjs.com/first-steps ; testing https://docs.nestjs.com/fundamentals/testing
- Docker and Compose: https://docs.docker.com/get-started/ ; https://docs.docker.com/compose/
- Caddy automatic HTTPS: https://caddyserver.com/docs/automatic-https ; how Let's Encrypt works
  https://letsencrypt.org/how-it-works/
- Cloudflare Workers static assets: https://developers.cloudflare.com/workers/static-assets/ ; DNS basics
  https://www.cloudflare.com/learning/dns/what-is-dns/
- GitHub Actions: https://docs.github.com/en/actions ; configuration as environment: https://12factor.net/config
- Optional course: The Missing Semester (shell, git, SSH) https://missing.csail.mit.edu/

**Concepts you will have used.** Monorepo boundaries, prerendering vs client rendering, schema migrations, RLS,
containers, reverse proxy and TLS, CI vs CD, secrets outside the repo, DNS delegation.

## Prototype session: ticket 05 (2 to 3 h, before Phase 2)

Resolve `issues/05` with the prototype skill: two or three throwaway variations of the player and results page in
one HTML file, react, record the direction in the ticket. Nothing from it is kept as code.

## Phase 1: Catalog, Question Bank and launch content (12 to 16 h plus 5 to 8 h of content review, weeks 3 to 6)

**Goal.** The content side of the data model exists with its rules, and about 300 reviewed questions are in the
production bank, produced by the real generator code run as a local command.

**Deliverables.** Migrations for `question`, `option`, `generation_batch`, `flag` (table only), enums, the option
shape check, `search` columns with GIN indexes, `topic_stock` and `question_stats` views, admin RLS via
`is_admin()` and `profile`; the `QuestionGenerator` port with the Anthropic and OpenAI adapters, strict JSON schema
in `libs/shared/domain`, versioned prompts, trigram de-duplication, verify pass, cost recording and caps; a CLI
entry in `apps/api` that runs a batch locally and writes drafts; the import endpoint logic as a module (HTTP
exposure comes in Phase 4) plus an import script; the blind bake-off on one topic and tier reviewed in a spreadsheet,
then generation for all 15 topic-tiers; review and approval offline, approval written by script.

**Exit test.** `select topic, tier, count(*) from topic_stock` shows at least 20 approved per topic and tier;
`select count(*) from question where state='approved' and kind='multi'` is at least 90; a pgTAP test proves anon
cannot read `option.is_correct`; a Vitest test proves a draft that fails the schema is retried once and then
counted as failed; the bake-off result and the chosen default adapter are recorded in `issues/07`.

**Read first.**
- Postgres constraints, enums and triggers: https://www.postgresql.org/docs/current/ddl-constraints.html ;
  https://www.postgresql.org/docs/current/plpgsql-trigger.html
- Full-text search and trigram similarity: https://www.postgresql.org/docs/current/textsearch.html ;
  https://www.postgresql.org/docs/current/pgtrgm.html
- Schema validation with zod: https://zod.dev
- Structured outputs: Anthropic https://docs.anthropic.com/en/docs/build-with-claude/structured-outputs ; OpenAI
  https://platform.openai.com/docs/guides/structured-outputs ; prompt caching on both vendors' docs
- Ports and adapters: https://alistair.cockburn.us/hexagonal-architecture/
- Evaluating LLM output: https://hamel.dev/blog/posts/evals/
- Optional course: Use The Index, Luke (indexes and query plans) https://use-the-index-luke.com/

**Concepts.** Relational modelling, check constraints, generated columns, text search, idempotent scripts, ports
and adapters, structured generation, cost accounting, blind evaluation.

## Phase 2: Quiz Play and the public trial (14 to 18 h, weeks 6 to 9)

**Goal.** Anyone can take one quiz at skillassert.com without an account, with the full settings, timer, results
and explanations; nothing trusted runs in the browser.

**Deliverables.** `attempt`, `attempt_question`, `bookmark` tables; `option_public` view; `start_quiz`,
`answer_question`, `submit_attempt`, `get_attempt_result` functions with pgTAP tests (trial rule, sampling by
format and source, no correctness leak, expiry, all-or-nothing grading); `event` writes for trial and quiz; anonymous
sign-in with Turnstile; settings screen with format availability from `topic_stock`; the player (SignalStore,
free navigation, overview grid, keyboard-first, timer from `expires_at`, bookmark and flag); results page per the
prototype direction with the sign-up prompt; light and dark themes; Playwright flow "trial quiz" including a
keyboard-only run.

**Exit test.** A fresh browser takes a timed mixed quiz end to end and sees explanations for missed questions; a
second start returns `trial_used`; `curl` to the REST endpoint for `option` with the anon key returns no
`is_correct` column; the Playwright trial flow passes in CI; Lighthouse accessibility on the player is 95 or more.

**Read first.**
- Postgres functions and SECURITY DEFINER: https://www.postgresql.org/docs/current/sql-createfunction.html ;
  https://supabase.com/docs/guides/database/functions ; RPC from the client
  https://supabase.com/docs/reference/javascript/rpc
- Angular signals and SignalStore: https://angular.dev/guide/signals ; https://ngrx.io/guide/signals/signal-store
- Routing and guards: https://angular.dev/guide/routing
- Material theming and CDK accessibility: https://material.angular.dev/guide/theming ;
  https://material.angular.dev/cdk/a11y/overview ; radio and checkbox group patterns
  https://www.w3.org/WAI/ARIA/apg/patterns/radio/
- Anonymous sign-ins and Turnstile: https://supabase.com/docs/guides/auth/auth-anonymous ;
  https://developers.cloudflare.com/turnstile/
- Database tests: https://pgtap.org/
- Optional course: the official Angular tutorials https://angular.dev/tutorials

**Concepts.** Trust boundaries, server-side grading, state machines in a store, zoneless change detection,
accessible forms, time handled by the server, bot protection.

## Phase 3: Identity and Learner pages, soft launch (14 to 18 h, weeks 9 to 12)

**Goal.** Visitors become learners: Google or magic-link sign-in, trial data carried over in both the sign-up and
returning-learner cases, bookmarks, notes, attempts, progress, data export, account deletion, and the legal pages
and safety net a public site with accounts needs.

**Deliverables.** Google provider, magic links, Mailjet SMTP with SPF, DKIM and DMARC on Cloudflare DNS; `profile`
trigger and `signup` and `trial_converted` events; anonymous upgrade with session refresh; `claim_trial` and
`merge_trial`; restrictive RLS for permanent-only data; `note` table with search; `learner_topic_progress` view;
`/me`, `/me/attempts`, `/me/bookmarks` with "Quiz me on these", `/me/notes`; "Retry what I missed" and the next-tier
nudge on results; `export_my_data`; `DELETE /me` on the API; `/privacy` and `/terms`; nightly `pg_dump` to Backblaze
B2 with a Better Stack heartbeat; Better Stack uptime on the web app and `/health`; Sentry in both apps; the daily
anonymous-user cleanup; Playwright flows "sign-up carry-over" and "returning learner merge".

**Exit test.** A trial taken logged out appears under `/me/attempts` after signing up with a magic link and,
separately, after signing in to an existing Google account; a magic-link email arrives from the sending subdomain
with DKIM pass; `export_my_data` returns the attempt; deleting the account removes every row; the backup object for
last night exists in B2 and the heartbeat is green; both Playwright flows pass in CI.

**Read first.**
- OAuth 2.0 and OpenID Connect: https://oauth.net/2/ ; https://openid.net/developers/how-connect-works/ ; JWTs
  https://jwt.io/introduction
- Supabase Auth: https://supabase.com/docs/guides/auth ; identity linking
  https://supabase.com/docs/guides/auth/auth-identity-linking ; passwordless email
  https://supabase.com/docs/guides/auth/auth-email-passwordless ; custom SMTP
  https://supabase.com/docs/guides/auth/auth-smtp
- Email authentication records: https://www.cloudflare.com/learning/dns/dns-records/dns-spf-record/ ;
  https://www.cloudflare.com/learning/dns/dns-records/dns-dkim-record/ ;
  https://www.cloudflare.com/learning/dns/dns-records/dns-dmarc-record/
- Authentication hygiene: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- Backups: https://www.postgresql.org/docs/current/app-pgdump.html ; B2 S3 API
  https://www.backblaze.com/docs/cloud-storage-s3-compatible-api
- Data protection basics for a small site: https://commission.europa.eu/law/law-topic/data-protection_en
- Optional course: PortSwigger Web Security Academy (authentication and access control labs)
  https://portswigger.net/web-security

**Concepts.** Federated identity, token refresh and claims, identity linking, proof of possession (the claim
token), restrictive policies, deliverability, backup and restore, data subject rights.

## Phase 4: Admin area and hosted pipeline (18 to 24 h, weeks 12 to 16)

**Goal.** Content runs from the product: generation batches from a form, review and approval in the queue,
categories managed in place, flags handled, the funnel visible.

**Deliverables.** NestJS JWT guard (JWKS) and admin guard; `POST /batches`, `GET /batches`, `GET /batches/:id`,
`POST /questions/import`, `POST /questions/bulk`; the batch worker on the `generation_batch` queue with three calls in
flight and cost caps; the `/admin` dashboard (stock, open flags, batches, funnel, question stats); the review queue
with search, filters, edit, approve and reject and flag resolution under admin RLS; the flag-count trigger and
three-flag pull; the categories editor; "Publish site" via `workflow_dispatch`; `funnel` view; Playwright flow
"admin approve".

**Exit test.** A batch of 20 submitted from the form appears in the queue within five minutes with verify notes and
a recorded cost under $0.50; an approved question is served by `start_quiz` and a rejected one is not; three flags
from three permanent learners pull a question and it shows first in the queue; the dashboard shows yesterday's
trials, submissions and sign-ups; the admin Playwright flow passes in CI.

**Read first.**
- NestJS guards, validation, configuration, scheduling: https://docs.nestjs.com/guards ;
  https://docs.nestjs.com/techniques/validation ; https://docs.nestjs.com/techniques/configuration ;
  https://docs.nestjs.com/techniques/task-scheduling
- Verifying Supabase JWTs: https://supabase.com/docs/guides/auth/jwts
- A database as a job queue (`FOR UPDATE SKIP LOCKED`):
  https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE
- Angular reactive forms and Material table: https://angular.dev/guide/forms/reactive-forms ;
  https://material.angular.dev/components/table/overview
- Triggering workflows from code: https://docs.github.com/en/rest/actions/workflows
- Optional course: none; the NestJS documentation read end to end is the course for this phase.

**Concepts.** Authorization at two layers, DTO validation, idempotent background jobs, back-pressure and caps,
audit of content changes, admin UX for queues.

## Phase 5: Operations, polish and launch (8 to 12 h, weeks 16 to 18)

**Goal.** The site is safe to leave running: monitored, backed up with a proven restore, measured, accessible, and
announced.

**Deliverables.** Monthly restore drill documented and run once; Cloudflare Web Analytics with the EU exclusion
off; Sentry alerts routed; uptime alerts routed; VPS hardening check (ufw, unattended upgrades, SSH keys only,
`ss -ltnp`); a11y audit on every learner screen with the keyboard-only Playwright run; dark-mode contrast pass;
prerender freshness check; the restyle from ticket 05 if it was deferred; a launch checklist and the public
announcement; `docs/runbook.md` with the ten things that can go wrong and what to do.

**Exit test.** The restore drill brings last night's dump into local Supabase and the category count matches
production; a forced error in the API appears in Sentry within a minute; stopping the API container pages you
within ten minutes; Lighthouse accessibility is 95 or more on home, settings, player, results, `/me`.

**Read first.**
- Sentry for Angular and NestJS: https://docs.sentry.io/platforms/javascript/guides/angular/ ;
  https://docs.sentry.io/platforms/javascript/guides/nestjs/
- Cloudflare Web Analytics: https://developers.cloudflare.com/web-analytics/
- Accessibility checks: https://www.w3.org/WAI/WCAG22/quickref/ ; https://playwright.dev/docs/accessibility-testing
- Monitoring that matters: https://sre.google/sre-book/monitoring-distributed-systems/
- SEO basics for the prerendered pages:
  https://developers.google.com/search/docs/fundamentals/seo-starter-guide
- Optional course: the Google SRE book, first eight chapters https://sre.google/sre-book/table-of-contents/

**Concepts.** Restore before backup counts, alerting on symptoms, runbooks, accessibility as a release gate.

## After v1

The v1.1 reserve from design.md section 2: revision history, quiz length option, learn-more links per topic,
feedback form and explanation thumbs, practice mode. Then the premium groundwork once there is traction: the
visibility column is already there.

## Tickets touched

05 before Phase 2; 07's bake-off result recorded in Phase 1; 09 closes when this plan is approved. New questions
during a phase go to a new ticket on the map, not into the phase plan.
