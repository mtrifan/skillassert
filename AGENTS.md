# AGENTS.md: skillassert

SkillAssert (skillassert.com) is a quiz web app and a personal learning project. These rules apply to every agent
and every machine. Sources of truth: `docs/planning/design.md` (design), `docs/planning/plan.md` (roadmap),
`docs/planning/plans/` (task plans with checkboxes), `GLOSSARY.md` (domain language), `docs/adr/` (decisions).

## Session protocol

1. `git pull --rebase` before anything else.
2. Read the last entry of `docs/planning/journal.md` and the next unchecked step in `docs/planning/plans/phase-*.md`.
3. One branch per task, cut from `main`, named `<type>/<slug>` (for example `feat/api-health`). Never work on the
   same task from two machines at once.
4. Test first: write the failing test, run it and see it fail for the right reason, implement, see it pass, then
   run the project's whole suite.
5. Tick the plan checkbox in the same commit as the work it describes.
6. Push before leaving the machine, even unfinished, as a draft pull request.
7. At the end of every session append an entry to `docs/planning/journal.md`: date, machine and tool, what was
   done, next step, gotchas.

## Git and delivery

- This is a personal project: no Jira, no ticket keys, branches from `main`, no work-specific workflows or tooling.
- Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`, `ci:`, `build:`, `test:`, `refactor:`), enforced by
  commitlint in CI.
- Everything reaches `main` through a pull request with green CI. Rebase, never merge commits.
- Secrets live only in GitHub Actions secrets and the VPS `.env`, never in the repository or in a chat. The Supabase
  project URL and anon key are public by design.

## Architecture rules

- The learner path never calls the NestJS API. Catalog, quizzes, attempts, bookmarks, flags, notes and auth go
  through Supabase under row-level security; starting a quiz, saving an answer and submitting are Postgres
  functions. The API owns only what needs a secret or bypasses learner permissions: generation, verify, import,
  bulk admin, account deletion, the health job. (ADR 0001, ADR 0002)
- Six modules with the same names on both sides: Catalog, Question Bank, Quiz Play, Learner, Identity, Admin. A
  module owns its tables; other modules reference them by id only.
- Supabase migrations in `supabase/migrations` own the schema; API tooling never generates migrations.

## Workspace layout

```
apps/web            Angular app (learner and admin areas)      apps/web-e2e   Playwright
apps/api            NestJS content service
libs/shared/domain  enums, types, zod schemas
libs/web/<module>/{feature,ui,data-access}   one slice per module
libs/web/ui         Angular Material wrappers, layout, theme
libs/web/data-access-supabase   the one place that creates the Supabase client
supabase/           config.toml, migrations/, seed.sql, tests/ (pgTAP)
infra/              docker-compose.prod.yml, Caddyfile, VPS bootstrap
docs/               planning/, adr/
```

## Nx tags and module boundaries

Every project carries one `scope:` tag (`scope:<module>`, `scope:shared`, `scope:web`, `scope:api`) and one `type:`
tag (`type:app`, `type:feature`, `type:ui`, `type:data-access`, `type:util`, `type:e2e`).
`@nx/enforce-module-boundaries` allows:

| Source | May depend on |
|---|---|
| `type:app` | `type:feature`, `type:ui`, `type:data-access`, `type:util` |
| `type:feature` | `type:feature`, `type:ui`, `type:data-access`, `type:util` |
| `type:ui` | `type:ui`, `type:util` |
| `type:data-access` | `type:data-access`, `type:util` |
| `type:util` | `type:util` |
| `scope:<x>` | `scope:<x>`, `scope:shared` |

A feature library uses its own module's ui and data-access and shared code, never another module's data-access.

## Angular conventions

Standalone components only, signals with signal inputs and outputs, OnPush, zoneless change detection, built-in
control flow, strict TypeScript, SCSS, element prefix `sa`, lazy routes per module. Every data-bound screen handles
loading, loaded, empty and error states.

## Toolchain

Node per `.nvmrc`, pnpm via corepack (`packageManager` in `package.json`). Common commands: `pnpm nx test <project>`,
`pnpm nx lint <project>`, `pnpm nx build web`, `pnpm nx e2e web-e2e`, `pnpm nx affected -t lint test build`.
