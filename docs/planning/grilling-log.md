# Grilling log

Rounds of the design interview. Each question carries a recommended answer; "yes" accepts it. Answers are recorded
under each round as given.

## Round 1 (2026-10-29): destination and the top-level branches

❓ **Q1** - **Destination and v1 scope**: The map's destination is an approved v1 design plus a phased build plan for skillassert.com. v1 = public homepage with categories and topics, quiz player with tiers 1 to 3 and an optional timer, results with explanations for missed questions, bookmarks and flags, a personal notes section, sign-in with Google and email, one anonymous trial quiz, an admin review panel, and an LLM pipeline that fills the question bank ahead of time. Premium, payments and user-posted quizzes stay out of v1 but the data model leaves room for them. "Scalable" means easy to add features through clean module boundaries, not built for heavy load on day one. Is that the destination?

➡️ Yes: everything listed is v1, nothing else is.

---

❓ **Q2** - **Backend philosophy**: Your two goals pull apart. "Ship a real site cheaply" favors a backend-as-a-service: Firebase or Supabase give you hosted DB, auth and storage with almost no server code. "Learn backend and infra" favors writing and deploying your own API. Three cuts:
- **A) BaaS only.** Fastest, least learning, most lock-in.
- **B) Own API + Postgres on a cheap VPS from day one.** Most learning, slowest; you carry TLS, backups and updates before you have a product.
- **C) Split by seam.** User-facing reads and writes go through a BaaS. The content pipeline, meaning LLM generation and admin bulk operations, is your own small service from day one, because it has to be server-side anyway: it holds the LLM key and writes past user permissions. Moving the rest onto your own API on a VPS becomes a planned learning phase later, not a rewrite.

➡️ C. My leaning for the BaaS is Supabase over Firebase: this domain is relational, SQL and row-level-security skills transfer anywhere, and Firestore's query limits make an admin "search all questions" panel painful. The concrete pick is confirmed in round 2, once research ticket 01 reports current free tiers and gotchas.

---

❓ **Q3** - **Backend language**, if Q2 is B or C: TypeScript with NestJS, or C# with ASP.NET Core? NestJS mirrors Angular with modules, DI and decorators, lets one Nx monorepo share types between frontend and API, and keeps you in one toolchain. C# with ASP.NET Core is the alternative if you want a second backend ecosystem, at the cost of learning it at the same time.

➡️ TypeScript with NestJS. Choose .NET only if learning that ecosystem is the single thing you most want from this project.

---

❓ **Q4** - **What a quiz is**: Model the content as a question bank. Each question belongs to one topic, for example Frontend › Angular, and one tier. A quiz attempt samples N approved questions for a topic and tier, optionally timed. Curated fixed sets such as "Angular interview set #3" can be layered on later without changing the model. Your "generate eagerly" and "review all questions by title or index" both fit this. Agreed?

➡️ Yes, with N = 10 per attempt as the default, my call, adjustable per topic.

---

❓ **Q5** - **Answer format**: "Multiple possible answers that the user can select" can mean single-choice or multi-select. Support both: each question declares its kind, options carry a correct mark, the player renders radio buttons or checkboxes accordingly, and a question scores all-or-nothing. Agreed?

➡️ Yes.

---

❓ **Q6** - **Content gate**: Every generated question is born as a draft and reaches users only after an admin approves it. The admin panel is first of all that approval queue: search by text or id, edit, approve, reject. A user flag sends an approved question back to review. Generation is triggered by the admin per topic and tier in batches, never at quiz start. Agreed?

➡️ Yes. Serving unreviewed LLM output is the fastest way to lose trust in a quiz site.

---

❓ **Q7** - **Anonymous trial**: Enforce "one quiz without sign-up" softly, with browser-local state plus an anonymous auth session, not device fingerprinting. When the visitor signs up, that attempt, its bookmarks and flags carry over to the new account. Agreed?

➡️ Yes. A hard limit costs more than the leak it closes, and carry-over is your conversion hook.

---

❓ **Q8** - **What premium means**: I read "premium tier quizzes that users can post" as a future paid plan that unlocks extra quizzes, some authored by users. v1 only reserves room: every quiz and question has an author, system or user, and a visibility, free or premium. No payments and no authoring UI in v1. Is that the right reading?

➡️ Yes.

---

❓ **Q9** - **Launch content**: English only, IT only. Five topics to start: Angular, React, TypeScript/JavaScript, backend fundamentals covering HTTP, REST and auth, and SQL. Three tiers each, named Novice, Intermediate, Senior. About 20 to 30 approved questions per topic and tier before launch, so roughly 300 to 450 questions through your review queue. Agreed?

➡️ Yes, starting with 20 per topic-tier and growing after launch.

---

❓ **Q10** - **Budget and cadence**: Running cost ceiling of about €10 per month for domain, a VPS or LLM credits, and €0 while everything fits free tiers. You can put in roughly 5 to 8 hours a week and want a first public slice live within about 6 weeks, then weekly increments. Right?

➡️ Yes. Correct the numbers if they are off; they size every phase.

---

❓ **Q11** - **Component library**: Angular Material plus the CDK. It is official, versioned in lockstep with Angular, and the CDK patterns transfer to any Angular codebase. PrimeNG is the free alternative if you want a bigger widget set over consistency with the Angular team's own components.

➡️ Yes, Angular Material.

### Answers (2026-10-29)

- Q1 yes. Q2 yes (C, split by seam). Q3 TypeScript with NestJS. Q4 yes. Q7 yes. Q8 yes, premium only after traction and good feedback.
- Q5 yes, plus: at quiz start the learner chooses the answer format, single-choice or multi-select (sharpened in Q17).
- Q6 yes, plus: the admin panel must let the admin trigger generation with filters; generated questions are reviewed and then become "quiz candidates" (term challenged in Q18; request shape in Q19).
- Q9 yes; future domains beyond IT, e.g. History, Finance (tree shape in Q16).
- Q10 budget raised: up to EUR 20/month is fine; cadence as proposed.
- Q11 Angular Material + CDK.

## Round 2 (2026-10-29): structure, lifecycle, admin, learning

Provider, hosting, auth flows and email delivery wait for research ticket 01; LLM provider waits for ticket 02.
Everything below is answerable now.

❓ **Q12** - **Repo and workspace**: One GitHub repository holding an Nx monorepo: `apps/web` for Angular, `apps/api` for the NestJS content service, `libs/shared` for types and validation schemas both sides import, and Angular libraries sliced per feature. This mirrors your work setup, teaches Nx boundaries for real, and makes shared DTOs a fact instead of a convention. Public repository: it doubles as a portfolio, and secrets never live in it anyway. Agreed?

➡️ Yes: Nx monorepo, public GitHub repo.

---

❓ **Q13** - **Angular baseline and state**: Latest stable Angular, standalone components only, signals for component state with signal inputs and outputs, OnPush everywhere, zoneless change detection, built-in control flow, lazy-loaded routes per feature, strict TypeScript. For state: plain services exposing signals by default, and NgRx SignalStore only for the quiz player, whose state is a real machine: timer, current question, answers, bookmarks, flags, submission. Agreed?

➡️ Yes. You learn both patterns and where each one earns its place.

---

❓ **Q14** - **Module map**: The "scalable" part is a fixed set of modules with explicit boundaries, same names on both sides of the stack: **Catalog** (categories, topics), **Question Bank** (questions, options, explanations, generation, review), **Quiz Play** (quizzes, attempts, scoring), **Learner** (bookmarks, flags, notes), **Identity** (accounts, roles, anonymous upgrade), **Admin** (the review and generation UI, built on the others). A new feature lands inside one of these or becomes a new module; no module reaches into another one's tables or internals. Agreed?

➡️ Yes.

---

❓ **Q15** - **Admin placement**: The admin panel is a lazy-loaded `/admin` area inside the same Angular app, behind an admin-role guard, with its own layout. A separate app only if bundle size or exposure ever becomes a real problem. Agreed?

➡️ Yes.

---

❓ **Q16** - **Category tree**: You said categories and subcategories, and later domains like History and Finance. Model one category tree of any depth, for example IT › Frontend › Angular or History › Ancient Rome. Questions attach only to leaf categories, which the glossary calls topics. The homepage shows the top level and lets the learner drill down. Agreed?

➡️ Yes.

---

❓ **Q17** - **Quiz format at start**: Your Q5 note: at quiz start the learner picks the format. Make it three options, single-choice, multi-select or mixed, with mixed as the default, and generation keeps both kinds stocked per topic and tier. Agreed?

➡️ Yes.

---

❓ **Q18** - **Question lifecycle and flags**: Every question is in exactly one state: Draft after generation, Approved after your review, or Rejected. Approved questions are the question bank, what you called quiz candidates; the glossary uses Approved and question bank so "candidate" never collides with the people taking quizzes. A flag does not pull a question from the bank by itself: it places the question in your review queue sorted by flag count, and only three flags from distinct learners pull it automatically, so one troll cannot empty the bank. Agreed?

➡️ Yes.

---

❓ **Q19** - **Generation request**: In the admin panel you fill a form: topic, tier, kind, how many questions, and an optional focus line such as "signals and effects" or "avoid NgModules". The batch runs in the background on the NestJS service, and its drafts appear in the review queue tagged with the batch. Before you see them, a second, stronger model pass checks each draft for answer correctness and marks doubtful ones, so you review less. Agreed?

➡️ Yes.

---

❓ **Q20** - **LLM provider**: Provider and models get picked by cost and structured-output quality from research ticket 02, not from memory. Do you have an existing API account or credits with Anthropic, OpenAI or Google that should tip the choice?

➡️ No existing credits; decide from the research.

---

❓ **Q21** - **Timer and scoring**: The timer is per quiz, not per question: duration = question count × a per-tier allowance, for example 60, 90 and 120 seconds per question for Novice, Intermediate and Senior; when it runs out the quiz submits itself. Score = correct questions out of total. No pass mark in v1. Agreed?

➡️ Yes.

---

❓ **Q22** - **Notes**: Notes are markdown text with a title, optionally linked to one question or one topic, listed newest first with a text search. Created from the results page, from a bookmarked question, or from the notes page itself. Agreed?

➡️ Yes.

---

❓ **Q23** - **Learning material format**: Each phase of the plan opens with a "Read first" list: official docs first, one deep-dive article per concept, at most three links per concept, plus one optional longer course per phase for the big topics such as Postgres and auth. Delivered in the plan file, not in chat. Agreed?

➡️ Yes.

---

❓ **Q24** - **Domain**: skillassert.com is registered and delegated to Porkbun nameservers (checked 2026-10-29 via DNS; the A records point at Porkbun parking). Is that your Porkbun account, so DNS is yours to change when we reach the hosting phase?

➡️ Yes.

---

❓ **Q25** - **Testing and CI**: Vitest for unit tests on both apps, Playwright for a handful of end-to-end flows (trial quiz, sign-up carry-over, admin approve), GitHub Actions running lint, test and build on every pull request. Agreed?

➡️ Yes.

### Answers (2026-10-29)

- Q12 to Q25 all accepted as recommended.
- Q17 extended: quiz settings at start also include timed vs untimed (format and timer are both learner choices).
- Q20: the user has a Codex (ChatGPT) subscription, open to another vendor if more economical (sharpened in Q35).

## Round 3 (2026-10-29): stack, data, operations

Built on rounds 1 and 2 and research ticket 01. The LLM vendor waits for research ticket 02; only the principle is
asked here (Q35).

❓ **Q26** - **The stack**: Supabase Free in an EU region for Postgres, Auth, row-level security and Storage. Your NestJS content service in Docker on a small EU VPS behind Caddy for automatic TLS: Hetzner first if an entry plan is orderable, OVH VPS-1 as the fallback, both under about €7 a month. Angular on Cloudflare Pages with Cloudflare DNS, which means moving the nameservers from Porkbun to Cloudflare; Porkbun stays the registrar. Everything is designed to stay on Supabase Free; the growth exit is self-hosting Supabase on a bigger VPS, not the $25 Pro plan. Agreed?

➡️ Yes.

---

❓ **Q27** - **Environments**: Local development runs Supabase on your machine through its CLI and Docker, with every schema change written as a migration file in the repo. One cloud project is production. Migrations reach production from CI, never by hand. No staging project in v1. Agreed?

➡️ Yes.

---

❓ **Q28** - **Auth and email**: Supabase Auth with Google, email plus password with mandatory email confirmation, and anonymous sign-ins. Manual identity linking is enabled so the anonymous session upgrades into the real account. A spike verifies, before the design is final, that the user id and its rows survive the upgrade, because the docs do not say. Supabase's default mailer is unusable in production, so a free transactional email provider with an EU option is wired in from day one; which one is a research ticket. Agreed?

➡️ Yes.

---

❓ **Q29** - **Where logic lives**: The learner path talks to Supabase directly: catalog reads, bookmarks, flags and notes through tables under row-level security, and the two operations that must not trust the browser, starting a quiz and submitting it, as Postgres functions that sample questions without exposing correct options, grade server-side, and reveal explanations only for a submitted attempt. The NestJS service owns generation, the verify pass, batch import and bulk admin operations, connects with a server-only role that bypasses row-level security, and checks the admin role from the Supabase token. Agreed?

➡️ Yes.

---

❓ **Q30** - **Core data model**: category (self-referencing tree); question (topic, tier, kind, prompt, explanation, state, author, visibility, batch); option (text, correct mark, order); generation_batch (filters, focus, status, model, cost); attempt (learner, quiz settings, timing, score) with attempt_question rows (position, selected options, correctness); bookmark; flag (one per learner and question, with a reason); note (title, markdown body, optional question or topic); profile (display name, role). A stored quiz entity arrives only with curated sets. Column detail goes in the design document. Agreed?

➡️ Yes.

---

❓ **Q31** - **Delivery pipeline**: Single `main` branch, pull requests with Conventional Commits. On every PR GitHub Actions runs lint, tests and build. On merge: Cloudflare Pages deploys the Angular app, the API image is built, pushed to the GitHub container registry and pulled on the VPS over SSH with Docker Compose, and pending Supabase migrations are applied. Agreed?

➡️ Yes.

---

❓ **Q32** - **Operations baseline**: Nightly logical database dump from the VPS to a free object store with two weeks of retention, because Supabase Free has no backups. A daily health job in the API that touches the database, which also keeps the Free project from pausing. One free uptime monitor and one free error tracker for both apps. The exact providers come from a research ticket. Agreed?

➡️ Yes.

---

❓ **Q33** - **Public pages**: Home and category pages are prerendered at build time with Angular's SSR tooling so search engines and link previews see real content; the rest stays a client-side app. No runtime SSR server. Agreed?

➡️ Yes.

---

❓ **Q34** - **Legal baseline**: Privacy policy and terms pages, an account deletion flow, and cookieless analytics such as Cloudflare Web Analytics so no cookie banner is needed in v1. Agreed?

➡️ Yes.

---

❓ **Q35** - **LLM access**: Your Codex subscription covers the Codex coding agent, not OpenAI API tokens; the generation service needs an API key billed per token, whichever vendor wins. One pipeline serves both the launch batch and ongoing generation, and the vendor and models are chosen from research 02 by cost and structured-output support. Codex stays your coding assistant. Agreed?

➡️ Yes.

### Answers (2026-10-29)

- Q26 to Q35 all accepted as recommended. Charting complete.

## Round 4 (2026-10-29): LLM vendor and generation pipeline (ticket 07)

Facts from research 02: generation costs cents to a few dollars at launch on any vendor; Anthropic and OpenAI
small and mid models are priced identically; Gemini's free tier is barred for apps offered in the EEA and its
caching does not fit our prompt size; reasoning tokens are billed as output and on by default. The pipeline sends
topic briefs, existing question prompts and drafts to the vendor, never learner data, so EU residency is not a
constraint for it.

❓ **Q36** - **Your subscription**: Which ChatGPT plan carries your Codex access? OpenAI's docs say Pro and Business subscribers can claim up to $500 a month of API credits without a payment method; Plus is not listed. If yours qualifies, the OpenAI API costs you nothing for this project and the vendor question is settled: gpt-6-luna drafts, gpt-6.1-sol verifies.

➡️ Tell me the plan. If it is Pro or Business, claim the credits and OpenAI is the vendor; if it is Plus, Q37 decides.
(Refined after the research update: Pro gets $100, $200 or $500 a month by tier, Business per seat; Plus, Go, Free
and Enterprise get nothing; the plan must be 7 days old and share its email with the API organization.)

---

❓ **Q37** - **Vendor without credits**: On price Anthropic and OpenAI tie, and no quality comparison exists for this task. I am Claude, so a recommendation from me is not neutral. Proposal: the generator port gets two adapters, Anthropic (claude-haiku-5-5 draft, claude-sonnet-5-5 verify) and OpenAI (gpt-6-luna draft, gpt-6.1-sol verify); the first batch for one topic and tier runs 20 questions through each, you review them blind in the queue, and the winner becomes the default while the other stays as a fallback. Total cost under a dollar, and it doubles as your first LLM evaluation exercise. Agreed?

➡️ Yes, unless Q36 already settled it.

---

❓ **Q38** - **Request shape and output discipline**: Five questions per request with a strict JSON schema (prompt, kind, 3 to 6 options with correct marks, an 80 to 150 word explanation, tags, one line on why this tier); reasoning effort at the lowest setting for drafting and low for verifying; a hard output-token limit per request; the long stable part of the prompt first so it caches on either vendor, the list of existing prompts to avoid last. Agreed?

➡️ Yes.

---

❓ **Q39** - **Synchronous, not batch API**: Calls run synchronously with three in flight, so a batch of 20 finishes in minutes and the admin sees progress; the vendors' 50% batch APIs, with up to 24 hours of turnaround, are not worth it at these costs. Agreed?

➡️ Yes.

---

❓ **Q40** - **Cost guardrails**: A per-batch cap, default $2, and a monthly cap, default $10, both enforced in the service from recorded usage, plus a spend limit set in the vendor console. A batch that would cross a cap refuses to start. Agreed?

➡️ Yes.

### Answers (2026-10-29)

- Q36: ChatGPT Plus, so no API credits. Q37 applies: two adapters, blind bake-off on the first batch.
- Q37, Q39 accepted. Q38 accepted after explanation (five questions per LLM call, configurable). Q40 accepted after
  explanation (caps apply to generation batches: per batch, per month, plus a vendor-console backstop).

## Round 5 (2026-10-29): builder-check questions only the owner can answer (ticket 08)

The fresh-eyes builder check raised seven questions. Four were settled in design.md as my calls: question text is
markdown with fenced code blocks; anonymous flags never count toward the pull rule; prerendered pages carry only
the category tree while counts load live, with an admin "Publish site" deploy hook; the prototype (ticket 05) is
not made yet, so the player and results may start on plain Material. Three change what gets built:

❓ **Q41** - **First public slice**: At 5 to 8 hours a week, the full stack does not fit in six weeks. Ship the learner path first: the Angular app on Supabase with launch content that you reviewed offline, generated by the same generator code run as a local command, read by you in the JSON or a spreadsheet, and imported with a script. The admin area and the hosted NestJS pipeline follow in the phases right after. The anti-goal still holds: you review every question, just not in the admin UI yet. Agreed?

➡️ Yes.

---

❓ **Q42** - **Launch stock per kind**: "About 20 approved per topic and tier" is read as 20 in total, generated at roughly 60/40 single to multi, so about 300 questions pass your review before launch. Mixed quizzes are available once a topic and tier has 10 approved questions, single-only once it has 10 single, multi-only when it reaches 10 multi, usually after launch; the settings screen shows what is available. The alternative, 20 per kind, means about 600 questions to review first. Agreed?

➡️ Yes: 20 in total, formats unlock by stock.

---

❓ **Q43** - **Trials by returning learners**: When someone who already has an account takes a trial while logged out and then signs in, the upgrade path cannot apply, because the identity already belongs to the account. Options: drop the trial with honest copy, or a claim-token merge: while anonymous, the client receives a claim token for its attempt; after sign-in it calls a merge function with that token, which re-keys the attempt, bookmarks and flags. A few hours in the Identity phase, and the same function covers the sign-up path if ticket 06 finds the user id does not survive conversion. The results-page copy becomes "Sign up or sign in to keep this result". Agreed?

➡️ Yes: claim-token merge in v1.

### Answers (2026-10-29)

- Q41, Q42, Q43 accepted as recommended.
