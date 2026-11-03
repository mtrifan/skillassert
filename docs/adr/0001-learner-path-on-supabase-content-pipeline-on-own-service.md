# 0001: Learner path on Supabase, content pipeline on our own service

Status: accepted (2026-10-29)

skillassert.com needs a hosted relational database with auth for a solo developer on a free budget, and the
developer wants to learn backend and infrastructure by doing. We split the backend by seam: everything a learner
touches (catalog, quizzes, attempts, bookmarks, flags, notes, sign-in) goes straight from the Angular app to
Supabase under row-level security, while the content pipeline (LLM generation, verification, import, bulk admin
operations) is our own NestJS service on a small VPS. The pipeline has to be server-side anyway because it holds the
LLM key and writes past learner permissions, so it is where the hand-written backend starts without putting the
learner path behind a box we operate ourselves.

## Considered options

- BaaS only (Firebase or Supabase, no own service): fastest, but the LLM batch does not fit any free serverless
  limit found (Supabase Edge 150 s, Workers 10 ms CPU, Vercel Hobby one cron a day), and it teaches little backend.
- Own API and Postgres on a VPS from day one: most learning, but TLS, backups and updates land before a product
  exists, and the learner path depends on one unmanaged box.
- Firebase instead of Supabase: rejected because the domain is relational, Firestore text search needs the
  Enterprise edition, and Cloud Functions require a billing account with no hard spend cap.

## Consequences

- Two runtimes to understand: Postgres with RLS and functions, and NestJS. Trusted quiz logic lives in Postgres
  (see ADR 0002), so the learner path works even when the VPS is down.
- We must stay on Supabase Free (Pro is above budget): design around the 500 MB database, the one-week pause rule,
  the unusable default mailer and the absence of backups.
- The growth exit is self-hosting Supabase on a bigger VPS or moving to Neon plus our own API; both keep Postgres.
