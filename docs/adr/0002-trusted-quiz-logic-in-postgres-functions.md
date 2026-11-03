# 0002: Trusted quiz logic lives in Postgres functions, not in the API

Status: accepted (2026-10-29)

Starting a quiz, saving answers and grading must not trust the browser: correct options and explanations may only
reach a learner after submission. Because the learner path talks to Supabase directly (ADR 0001), these operations
are SECURITY DEFINER Postgres functions exposed through Supabase RPC, and learners have no direct read access to the
column that marks an option correct. A reader coming from NestJS would expect API endpoints here; putting them in
SQL keeps the learner path independent of the VPS and teaches the database as a real application layer.

## Consequences

- SQL functions get the same treatment as application code: migrations, tests against a local Supabase, and reviews.
- The NestJS service never grades or serves quizzes; if that ever changes, this decision is being reversed.
