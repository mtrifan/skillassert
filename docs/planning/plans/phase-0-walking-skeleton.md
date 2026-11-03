# Phase 0: Walking Skeleton Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy the thinnest slice through every layer of skillassert.com: a prerendered Angular home page on Cloudflare Workers that lists the category tree from production Supabase, a NestJS `GET /health` that touches the database behind Caddy on an EU VPS, and migrations, image and web build all flowing from a merge to `main`.

**Architecture:** Nx monorepo (`apps/web`, `apps/api`, `libs/shared/domain`, two catalog libs) with enforced module boundaries. The learner path reads Supabase directly under row-level security; the API never sits on that path. Production is one Supabase project in Frankfurt, one VPS running Docker Compose with Caddy and the API, and one Worker serving static assets with a ten-line fallback to the client shell.

**Tech Stack:** Node 24 LTS (22.12+ acceptable), pnpm via corepack, Nx latest, Angular latest stable (standalone, signals, zoneless, SSR tooling for prerender), Angular Material, NestJS, `@supabase/supabase-js`, zod, `pg`, `@nestjs/schedule`, `@nestjs/config`, Vitest, Playwright, Supabase CLI with pgTAP, Docker, Caddy 2, Wrangler, GitHub Actions.

**Spec:** `docs/planning/design.md` (sections 4, 5, 6, 13, 16) and `docs/planning/plan.md` (Phase 0). Facts behind hosting choices: `research/01` and `research/04`.

## Global Constraints

- The learner path never calls the API (design section 4). The API container publishes no ports; only Caddy publishes 80 and 443 (design section 13).
- Secrets live only in GitHub Actions secrets and the VPS `.env` file, never in the repository. The Supabase project URL and anon key are public by design and may be committed in environment files.
- Conventional Commits; every change after the first commit reaches `main` through a pull request with green CI.
- EU only: Supabase region Frankfurt (`eu-central-1`); VPS in Germany or Finland (Hetzner `fsn1`, `nbg1`, `hel1`) or France (OVH Gravelines or Strasbourg). VPS price under EUR 7 a month; everything else on free tiers.
- Nx tags on every project: `scope:<module>|shared|web|api` and `type:app|feature|ui|data-access|util`; `@nx/enforce-module-boundaries` on.
- Angular app: standalone components only, zoneless change detection, strict TypeScript, SCSS, element prefix `sa`.
- Names: product "SkillAssert", domain `skillassert.com`, API `api.skillassert.com`, Worker `skillassert-web`, image `ghcr.io/<github-owner>/skillassert-api`.
- Database connection from the VPS uses the Supabase session pooler URL (IPv4), not the direct host, which is IPv6-only on the Free plan.

## Review Focus

1. Supabase Free pauses a project after a week of low database activity: the API's daily keep-alive job must touch the database. Pinned by the `KeepAliveJob` tests in Task 3.
2. Docker-published ports bypass ufw: the API must publish nothing and the VPS must listen only on 22, 80 and 443. Pinned by the `ss -ltnp` verification in Task 7.
3. Caddy certificates must survive container recreation, or Let's Encrypt rate limits bite after a few deploys. Pinned by the double `--force-recreate` check in Task 7.
4. Deep links on Workers must return the client shell and prerendered pages must be served as real HTML. Pinned by the two `curl` checks in Task 8 and the `pickFallback` tests.
5. Migrations run before the API and web deploys and are idempotent: re-running the deploy workflow changes nothing. Pinned by the `needs:` ordering and the re-run check in Task 9.

---

### Task 0: Accounts, domain and VPS (human checklist, no code)

**Files:** none in the repository. Keep ids and hosts in your password manager, never in the repo.

- [x] **Step 1: GitHub.** Create the public repository `skillassert`, empty (no README, licence or gitignore).
- [ ] **Step 2: Cloudflare zone.** Add `skillassert.com` on the Free plan; copy the two assigned nameservers; at Porkbun replace the authoritative nameservers with them. Verify: `dig NS skillassert.com +short` lists `*.ns.cloudflare.com` (allow up to a day).
- [ ] **Step 3: Cloudflare token.** Create an API token from the "Edit Cloudflare Workers" template; note the Account ID from the dashboard sidebar.
- [ ] **Step 4: Supabase.** Create an organization and the project `skillassert` in Frankfurt (`eu-central-1`); store the database password; copy the project URL, anon key and project ref from Project Settings; create a personal access token under Account > Access Tokens for CI.
- [ ] **Step 5: VPS.** Order Hetzner CAX11 or CX23 in `fsn1`, `nbg1` or `hel1` with Ubuntu 24.04 LTS and your SSH public key; if neither is orderable, OVH VPS-1 in Gravelines or Strasbourg. Note the IPv4 address. Verify: `ssh root@<ip> true` exits 0.
- [ ] **Step 6: DNS for the API.** In Cloudflare DNS add `A api -> <vps ip>`, proxy status DNS only (grey cloud). Verify: `dig api.skillassert.com +short` prints the IP.
- [ ] **Step 7: Supabase connection string.** From Project Settings > Database copy the session pooler URI (port 5432, host `aws-0-eu-central-1.pooler.supabase.com`); this is `DATABASE_URL` for the VPS later.

---

### Task 1: Nx workspace with the Angular app

**Files:**
- Create: workspace root (`nx.json`, `package.json`, `pnpm-lock.yaml`, `tsconfig.base.json`, eslint config), `apps/web/**`, `apps/web-e2e/**`, `.nvmrc`, `.gitignore`, `README.md`

**Interfaces:**
- Produces: project names `web` and `web-e2e`; app root selector `sa-root`; `apps/web/src/app/app.config.ts` with zoneless change detection.

- [x] **Step 1: Toolchain.** `corepack enable`; `node -v` prints 24.x (or 22.12+). Write `24` to `.nvmrc` after the workspace exists.
- [x] **Step 2: Generate the workspace.** From the parent folder (`cd ~/Developer/Personal`; the generator creates `~/Developer/Personal/skillassert`), run `npx create-nx-workspace@latest skillassert --preset=angular-monorepo --appName=web --style=scss --bundler=esbuild --ssr=true --e2eTestRunner=playwright --unitTestRunner=vitest --prefix=sa --packageManager=pnpm --nxCloud=skip`. If a flag is rejected, drop it and reach the same outcome with the generator's prompts. Required outcome: `apps/web` standalone app with SSR tooling, Vitest unit tests, Playwright e2e project, pnpm lockfile, git repository initialised.
- [x] **Step 3: Verify the generated baseline.** Run `pnpm nx build web`, `pnpm nx test web`, `pnpm nx e2e web-e2e`. Expected: all three pass.
- [x] **Step 4: Zoneless check.** `grep -rn "zone.js" apps/web/project.json apps/web/src/main.ts` prints nothing; `grep -n "provideZonelessChangeDetection" apps/web/src/app/app.config.ts` prints one line. If zone.js is present, remove the polyfill and add the provider.
- [x] **Step 5: Write the failing shell test** in `apps/web/src/app/app.spec.ts`:

```ts
it('renders the product name in the header', async () => {
  const fixture = TestBed.createComponent(App);
  await fixture.whenStable();
  expect(fixture.nativeElement.querySelector('header')?.textContent).toContain('SkillAssert');
});
```

- [x] **Step 6: Run it to verify it fails.** `pnpm nx test web`. Expected: FAIL, header missing.
- [x] **Step 7: Implement the shell.** Replace the generated welcome markup in `apps/web/src/app/app.html` with `<header>SkillAssert</header><main><router-outlet /></main>`; delete the generated `nx-welcome` component and its import.
- [x] **Step 8: Run tests to verify they pass.** `pnpm nx test web`. Expected: PASS.
- [x] **Step 9: Bring the planning workspace into the repository.** `mkdir -p docs && cp -R ~/.claude/skillassert docs/planning`; move `docs/planning/GLOSSARY.md` to the repo root and `docs/planning/docs/adr` to `docs/adr`; delete `docs/planning/docs`; in `docs/planning/README.md`, `plan.md` and `plans/*.md` replace every `~/.claude/skillassert/` with `docs/planning/`. Replace `~/.claude/skillassert/README.md` on this Mac with three lines pointing at the repository, so there is one source of truth.
- [x] **Step 10: Agent instructions for every tool.** Create `AGENTS.md` at the repo root: the module map and Nx tag rules (design section 5), the learner-path rule (section 4), Conventional Commits, test first, pull before starting and read the next unchecked step in `docs/planning/plans/`, one branch per task, append a journal entry to `docs/planning/journal.md` at the end of every session (date, machine and tool, what was done, next step, gotchas). Create `CLAUDE.md` containing only `@AGENTS.md`. Codex reads `AGENTS.md` natively.
- [x] **Step 11: Cross-platform hygiene.** `.gitattributes` with `* text=auto eol=lf`; `.editorconfig` already generated; `.nvmrc` present.
- [x] **Step 12: Engines and first commit.** Add `"engines": { "node": ">=22.12" }` to `package.json`. `git add -A && git commit --amend -m "chore(workspace): create nx workspace with the angular web app and planning docs"` (the generator's initial commit), then `git remote add origin git@github.com:<owner>/skillassert.git && git push -u origin main`. This is the only direct push to `main`.

---

### Task 2: Shared domain library and module boundaries

**Files:**
- Create: `libs/shared/domain/src/index.ts`, `libs/shared/domain/src/lib/health.ts`, `libs/shared/domain/src/lib/health.spec.ts`
- Modify: root eslint config (module boundary constraints), `apps/web/project.json` (tags)

**Interfaces:**
- Produces: `healthResponseSchema` (zod) and `type HealthResponse = { status: 'ok' | 'degraded'; db: 'ok' | 'error'; version: string }` exported from `@skillassert/shared/domain`; tag vocabulary `scope:*`, `type:*`.

- [ ] **Step 1: Generate.** `pnpm add zod`; `pnpm nx g @nx/js:library libs/shared/domain --bundler=none --unitTestRunner=vitest --importPath=@skillassert/shared/domain --tags=scope:shared,type:util`.
- [ ] **Step 2: Write the failing test** in `health.spec.ts`:

```ts
it('accepts a valid health response', () => {
  expect(healthResponseSchema.parse({ status: 'ok', db: 'ok', version: '0.0.0' })).toEqual({ status: 'ok', db: 'ok', version: '0.0.0' });
});
it('rejects an unknown status', () => {
  expect(() => healthResponseSchema.parse({ status: 'fine', db: 'ok', version: '0.0.0' })).toThrow();
});
```

- [ ] **Step 3: Run it to verify it fails.** `pnpm nx test shared-domain`. Expected: FAIL, `healthResponseSchema` not exported.
- [ ] **Step 4: Implement** `healthResponseSchema` with `z.object({ status: z.enum(['ok','degraded']), db: z.enum(['ok','error']), version: z.string() })` and `export type HealthResponse = z.infer<typeof healthResponseSchema>` in `health.ts`; re-export from `index.ts`.
- [ ] **Step 5: Run tests to verify they pass.** `pnpm nx test shared-domain`. Expected: PASS.
- [ ] **Step 6: Boundaries.** Tag `web` with `scope:web,type:app` and `web-e2e` with `scope:web,type:e2e`. In the root eslint config set `@nx/enforce-module-boundaries` `depConstraints`: `type:app` -> `type:feature|ui|data-access|util`; `type:feature` -> `type:feature|ui|data-access|util`; `type:ui` -> `type:ui|util`; `type:data-access` -> `type:data-access|util`; `type:util` -> `type:util`; every `scope:<x>` -> `scope:<x>|scope:shared`.
- [ ] **Step 7: Verify.** `pnpm nx run-many -t lint`. Expected: PASS. Then prove the rule bites: add `import { healthResponseSchema } from '@skillassert/shared/domain';` to `apps/web-e2e/src/example.spec.ts` after temporarily changing `web-e2e`'s tag to `type:util`; `pnpm nx lint web-e2e` must FAIL with an enforce-module-boundaries error; revert both edits.
- [ ] **Step 8: Commit.** `git switch -c chore/shared-domain`; `git add libs/shared/domain eslint.config.* apps/web/project.json package.json pnpm-lock.yaml`; `git commit -m "feat(shared): add domain library with health schema and module boundaries"`; push and open a pull request (CI arrives in Task 6; merge after review).

---

### Task 3: NestJS API with a database-backed health check and the daily keep-alive job

**Files:**
- Create: `apps/api/src/main.ts`, `apps/api/src/app/app.module.ts`, `apps/api/src/app/db/db-pinger.ts`, `apps/api/src/app/db/pg-db-pinger.ts`, `apps/api/src/app/health/health.service.ts`, `apps/api/src/app/health/health.service.spec.ts`, `apps/api/src/app/health/health.controller.ts`, `apps/api/src/app/health/health.controller.spec.ts`, `apps/api/src/app/keepalive/keepalive.job.ts`, `apps/api/src/app/keepalive/keepalive.job.spec.ts`, `apps/api/.env.example`
- Modify: `apps/api/project.json` (tags `scope:api,type:app`)

**Interfaces:**
- Consumes: `HealthResponse`, `healthResponseSchema` from Task 2.
- Produces: `DB_PINGER` injection token with `interface DbPinger { ping(): Promise<void> }`; `PgDbPinger` (a `pg.Pool` on `DATABASE_URL`, `select 1`); `APP_VERSION` token provided by a factory reading `ConfigService.get('APP_VERSION') ?? '0.0.0'`; `HealthService.check(): Promise<HealthResponse>`; `GET /health` returning 200 with the body on `ok` and 503 on `degraded`; `KEEPALIVE_CRON = '0 3 * * *'` and `KeepAliveJob.run(): Promise<void>`.

- [ ] **Step 1: Generate.** `pnpm add @nestjs/config @nestjs/schedule pg` and `pnpm add -D @types/pg supertest @types/supertest`; `pnpm nx g @nx/nest:application apps/api --e2eTestRunner=none --unitTestRunner=vitest`. If the generator refuses Vitest, accept Jest and note the deviation in design.md section 13.
- [ ] **Step 2: Write the failing service tests** in `health.service.spec.ts`:

```ts
it('reports ok when the database answers', async () => {
  const service = new HealthService({ ping: async () => undefined }, '1.2.3');
  await expect(service.check()).resolves.toEqual({ status: 'ok', db: 'ok', version: '1.2.3' });
});
it('reports degraded when the database fails', async () => {
  const service = new HealthService({ ping: async () => { throw new Error('down'); } }, '1.2.3');
  await expect(service.check()).resolves.toEqual({ status: 'degraded', db: 'error', version: '1.2.3' });
});
```

- [ ] **Step 3: Run them to verify they fail.** `pnpm nx test api`. Expected: FAIL, `HealthService` missing.
- [ ] **Step 4: Implement** `HealthService` (`@Injectable()`, constructor `(@Inject(DB_PINGER) pinger: DbPinger, @Inject(APP_VERSION) version: string)`), `DB_PINGER`, `APP_VERSION` tokens and `PgDbPinger` in the files above. The version comes from `ConfigService.get('APP_VERSION') ?? '0.0.0'`.
- [ ] **Step 5: Run tests to verify they pass.** `pnpm nx test api`. Expected: PASS.
- [ ] **Step 6: Write the failing controller tests** in `health.controller.spec.ts` with `Test.createTestingModule` overriding `DB_PINGER`, and supertest against `app.getHttpServer()`:

```ts
it('GET /health returns 200 and a valid body when ok', async () => {
  const res = await request(server).get('/health').expect(200);
  expect(healthResponseSchema.parse(res.body).status).toBe('ok');
});
it('GET /health returns 503 when the database fails', async () => {
  const res = await request(serverWithFailingDb).get('/health').expect(503);
  expect(res.body.db).toBe('error');
});
```

- [ ] **Step 7: Run them to verify they fail.** Expected: FAIL, route missing.
- [ ] **Step 8: Implement** `HealthController` with `@Get('health')` that calls `check()` and sets status 503 through `@Res({ passthrough: true })` when degraded.
- [ ] **Step 9: Run tests to verify they pass.** `pnpm nx test api`. Expected: PASS.
- [ ] **Step 10: Write the failing job tests** in `keepalive.job.spec.ts`:

```ts
it('runs at 03:00 UTC daily', () => { expect(KEEPALIVE_CRON).toBe('0 3 * * *'); });
it('touches the database through the health check', async () => {
  const check = vi.fn().mockResolvedValue({ status: 'ok', db: 'ok', version: 'x' });
  await new KeepAliveJob({ check } as unknown as HealthService).run();
  expect(check).toHaveBeenCalledTimes(1);
});
```

- [ ] **Step 11: Run them to verify they fail.** `pnpm nx test api`. Expected: FAIL, `KeepAliveJob` missing.
- [ ] **Step 11b: Implement** `KeepAliveJob` with `@Cron(KEEPALIVE_CRON)` on `run()`, and register `ScheduleModule.forRoot()` plus `ConfigModule.forRoot({ isGlobal: true })` in `AppModule`.
- [ ] **Step 11c: Run tests to verify they pass.** `pnpm nx test api`. Expected: PASS.
- [ ] **Step 12: Environment example.** `apps/api/.env.example` with `DATABASE_URL=postgresql://postgres:postgres@127.0.0.1:54322/postgres`, `PORT=3000`, `APP_VERSION=dev`. Confirm `.env` is in `.gitignore`.
- [ ] **Step 13: Commit.** Branch `feat/api-health`; `git add apps/api libs package.json pnpm-lock.yaml`; `git commit -m "feat(api): add health endpoint with database ping and daily keep-alive job"`; pull request.

---

### Task 4: Local Supabase, first migrations, RLS and seed

**Files:**
- Create: `supabase/config.toml` (generated), `supabase/migrations/<timestamp>_create_category.sql`, `supabase/migrations/<timestamp>_seed_launch_categories.sql`, `supabase/seed.sql` (local-only data, empty for now), `supabase/tests/category.test.sql`
- Modify: `.gitignore` (`supabase/.temp`), root `package.json` scripts `db:start`, `db:reset`, `db:test`

**Interfaces:**
- Produces: table `public.category(id uuid pk default gen_random_uuid(), parent_id uuid null references category(id) on delete restrict, slug text not null, name text not null, description text null, sort_order int not null default 0, is_published boolean not null default false, created_at timestamptz not null default now(), updated_at timestamptz not null default now(), unique nulls not distinct (parent_id, slug))`; RLS enabled; policy `category_read_published` for roles `anon, authenticated` `using (is_published)`; no write policies yet. Launch tree (all published, slugs in parentheses): IT (`it`) > Frontend (`frontend`) > Angular (`angular`), React (`react`); IT > Languages (`languages`) > TypeScript and JavaScript (`typescript-javascript`); IT > Backend (`backend`) > Fundamentals (`fundamentals`); IT > Data (`data`) > SQL (`sql`). Ten rows.

- [ ] **Step 1: Install and start.** `brew install supabase/tap/supabase`; `supabase init` at the repo root; `supabase start`. Expected: local API on `127.0.0.1:54321`, database on `54322`, Studio URL printed.
- [ ] **Step 2: Write the failing database test** `supabase/tests/category.test.sql`:

```sql
begin; select plan(4);
select has_table('public', 'category');
insert into public.category (slug, name, is_published) values ('history', 'History', false);
set local role anon;
select is((select count(*)::int from public.category where is_published), 10, 'anon sees the ten launch rows');
select is((select count(*)::int from public.category where slug = 'history'), 0, 'anon cannot see unpublished rows');
select throws_ok($$insert into public.category (slug, name) values ('x', 'x')$$, '42501', null, 'anon cannot insert');
select * from finish(); rollback;
```

- [ ] **Step 3: Run it to verify it fails.** `supabase test db`. Expected: FAIL, table missing.
- [ ] **Step 4: Write the migrations.** `supabase migration new create_category` with the table, `alter table ... enable row level security` and the policy; `supabase migration new seed_launch_categories` inserting the ten rows with `on conflict (parent_id, slug) do nothing` (idempotent, so it is safe on every `db push`). Apply with `supabase db reset`.
- [ ] **Step 5: Run the test to verify it passes.** `supabase test db`. Expected: 4 of 4 pass.
- [ ] **Step 6: Health against a real database.** `cp apps/api/.env.example apps/api/.env`; `pnpm nx serve api`; `curl -s localhost:3000/health`. Expected: `{"status":"ok","db":"ok","version":"dev"}`.
- [ ] **Step 7: Scripts and commit.** Add `"db:start": "supabase start"`, `"db:reset": "supabase db reset"`, `"db:test": "supabase test db"`. Branch `feat/db-category`; `git add supabase package.json .gitignore`; `git commit -m "feat(db): add category table with row-level security and the launch tree"`; pull request.

---

### Task 5: Home page renders the category tree, prerendered

**Files:**
- Create: `libs/web/data-access-supabase/src/lib/supabase-client.ts` (+ `index.ts`), `libs/web/catalog/data-access/src/lib/category.gateway.ts`, `libs/web/catalog/data-access/src/lib/build-tree.ts`, `libs/web/catalog/data-access/src/lib/build-tree.spec.ts`, `libs/web/catalog/feature/src/lib/home/home.component.ts|html|spec.ts`, `apps/web/src/environments/environment.ts` and `environment.production.ts`, `apps/web-e2e/src/home.spec.ts`
- Modify: `apps/web/src/app/app.routes.ts`, `apps/web/src/app/app.routes.server.ts`, `apps/web/project.json` (build options, file replacements, prerender)

**Interfaces:**
- Produces: `SUPABASE_CLIENT` injection token providing `SupabaseClient` from `createClient(environment.supabaseUrl, environment.supabaseAnonKey)`; `type CategoryRow = { id: string; parent_id: string | null; slug: string; name: string; sort_order: number }`; `type CategoryNode = { id: string; slug: string; name: string; children: CategoryNode[] }`; `buildTree(rows: CategoryRow[]): CategoryNode[]` (roots first, children ordered by `sort_order` then `name`); `CATEGORY_GATEWAY` token with `interface CategoryGateway { listPublishedTree(): Promise<CategoryNode[]> }` and `SupabaseCategoryGateway` selecting `id, parent_id, slug, name, sort_order` from `category`; `HomeComponent` (selector `sa-home`).

- [ ] **Step 1: Generate libs and install.** `pnpm add @supabase/supabase-js @angular/material @angular/cdk`; `pnpm nx g @nx/angular:library libs/web/data-access-supabase --tags=scope:shared,type:data-access --standalone`; `pnpm nx g @nx/angular:library libs/web/catalog/data-access --tags=scope:catalog,type:data-access --standalone`; `pnpm nx g @nx/angular:library libs/web/catalog/feature --tags=scope:catalog,type:feature --standalone`. Run the Material schematic for the app, `pnpm nx g @angular/material:ng-add --project=web`, choosing a prebuilt theme and the Roboto font; proper theming and `prefers-color-scheme` arrive in Phase 2.
- [ ] **Step 2: Write the failing tree test** in `build-tree.spec.ts`:

```ts
it('nests children under parents ordered by sort_order then name', () => {
  const rows = [
    { id: 'it', parent_id: null, slug: 'it', name: 'IT', sort_order: 0 },
    { id: 'fe', parent_id: 'it', slug: 'frontend', name: 'Frontend', sort_order: 1 },
    { id: 'ng', parent_id: 'fe', slug: 'angular', name: 'Angular', sort_order: 0 },
    { id: 'data', parent_id: 'it', slug: 'data', name: 'Data', sort_order: 0 },
  ];
  expect(buildTree(rows)).toEqual([
    { id: 'it', slug: 'it', name: 'IT', children: [
      { id: 'data', slug: 'data', name: 'Data', children: [] },
      { id: 'fe', slug: 'frontend', name: 'Frontend', children: [ { id: 'ng', slug: 'angular', name: 'Angular', children: [] } ] },
    ] },
  ]);
});
```

- [ ] **Step 3: Run it to verify it fails,** implement `buildTree`, run again: PASS. `pnpm nx test web-catalog-data-access`.
- [ ] **Step 4: Gateway and client.** Implement `SUPABASE_CLIENT`, `CATEGORY_GATEWAY`, `SupabaseCategoryGateway.listPublishedTree()` (query, throw on `error`, return `buildTree(data)`); provide both in `app.config.ts`.
- [ ] **Step 5: Write the failing component tests** in `home.component.spec.ts` with a fake gateway (`vi.fn()`): loading state shows the text `Loading topics`; loaded state renders one link per node with `href="/c/<slug>"` and nested lists; empty array shows `No topics yet`; a rejected promise shows `Could not load topics`.
- [ ] **Step 6: Run them to verify they fail,** implement `HomeComponent` with a signal `state: 'loading' | 'loaded' | 'empty' | 'error'` and `tree` signal (use Angular's `resource()` if the installed version has it stable, else set the signals from the promise in the constructor), template with `@switch` and a recursive `@for`. Run: PASS. `pnpm nx test web-catalog-feature`.
- [ ] **Step 7: Routes and prerender.** `app.routes.ts`: `''` lazy-loads `HomeComponent`. `app.routes.server.ts`: `''` -> `RenderMode.Prerender`, `'**'` -> `RenderMode.Client`. In `apps/web/project.json` build options set `"outputMode": "static"` with file replacement of `environment.ts` by `environment.production.ts` in the `production` configuration; `environment.ts` points at `http://127.0.0.1:54321` with the local anon key printed by `supabase start`, `environment.production.ts` at the production URL and anon key from Task 0.
- [ ] **Step 8: Verify the prerender locally.** With `supabase start` running: `pnpm nx build web --configuration=development`; `ls dist/apps/web/browser` shows `index.html` and `index.csr.html`; `grep -c Angular dist/apps/web/browser/index.html` prints 1 or more.
- [ ] **Step 9: Write the failing e2e** `apps/web-e2e/src/home.spec.ts`: `await page.goto('/'); await expect(page.getByRole('link', { name: 'Angular' })).toBeVisible();`. Run `pnpm nx e2e web-e2e`: it should already pass against the local stack; if it fails, fix the app, not the test.
- [ ] **Step 10: Commit.** Branch `feat/catalog-home`; add the new libs, app changes and e2e; `git commit -m "feat(catalog): render the published category tree on a prerendered home page"`; pull request.

---

### Task 6: CI on pull requests

**Files:**
- Create: `.github/workflows/ci.yml`, `commitlint.config.js`
- Modify: `package.json` (dev deps `@commitlint/cli`, `@commitlint/config-conventional`)

**Interfaces:**
- Produces: required checks `ci / verify` and `ci / commitlint` on `main`.

- [ ] **Step 1: Workflow.** `ci.yml` on `pull_request` to `main`: job `verify` on `ubuntu-latest`: `actions/checkout@v4` with `fetch-depth: 0`; `actions/setup-node@v4` with the `.nvmrc` version and `corepack enable`; `pnpm install --frozen-lockfile`; `supabase/setup-cli@v1`; `supabase start`; `supabase test db`; `pnpm exec playwright install --with-deps chromium`; `pnpm nx affected -t lint test build e2e --base=origin/main --head=HEAD`. Job `commitlint`: `pnpm exec commitlint --from origin/main --to HEAD --verbose` with `commitlint.config.js` extending `@commitlint/config-conventional`.
- [ ] **Step 2: Verify red.** Push the branch with a deliberately bad commit message (`wip`) and open the pull request. Expected: `commitlint` fails, `verify` passes.
- [ ] **Step 3: Verify green.** `git commit --amend -m "ci: run lint, tests, build, database tests and e2e on pull requests"`; force-push the branch. Expected: both checks pass.
- [ ] **Step 4: Protect main.** In GitHub settings require pull requests and the two checks before merging. Merge this pull request and the earlier ones from Tasks 2 to 5 in order, rebasing each on `main`.

---

### Task 7: API container, Caddy, VPS bootstrap and roll-out on main

**Files:**
- Create: `apps/api/Dockerfile`, `.dockerignore`, `infra/docker-compose.prod.yml`, `infra/Caddyfile`, `infra/vps-bootstrap.sh`, `.github/workflows/deploy.yml` (job `api`)

**Interfaces:**
- Produces: image `ghcr.io/<owner>/skillassert-api:<git sha>` and `:latest`, non-root, `CMD ["node", "dist/apps/api/main.js"]`, listening on 3000; Compose services `caddy` (image `caddy:2`, ports `80:80`, `443:443`, volumes `caddy_data:/data`, `caddy_config:/config`, `./Caddyfile:/etc/caddy/Caddyfile`) and `api` (image above, `env_file: .env`, `restart: unless-stopped`, no `ports`); `Caddyfile`: `api.skillassert.com { reverse_proxy api:3000 }`; VPS layout `/opt/skillassert/{docker-compose.yml,Caddyfile,.env}` owned by user `deploy`.

- [ ] **Step 1: Dockerfile.** Multi-stage: stage 1 `node:24-alpine` with corepack, `pnpm install --frozen-lockfile`, `pnpm nx build api --configuration=production`; stage 2 `node:24-alpine`, `USER node`, copy `dist/apps/api` and production `node_modules` (`pnpm deploy` or `pnpm install --prod` against the built package), `ENV PORT=3000`, `EXPOSE 3000`. `.dockerignore` excludes `node_modules`, `dist`, `.git`, `supabase/.temp`.
- [ ] **Step 2: Verify the image locally.** `docker build -f apps/api/Dockerfile -t skillassert-api .`; `docker run --rm -e DATABASE_URL=postgresql://postgres:postgres@host.docker.internal:54322/postgres -e APP_VERSION=local -p 3000:3000 skillassert-api`; `curl -s localhost:3000/health`. Expected: `{"status":"ok","db":"ok","version":"local"}`.
- [ ] **Step 3: Bootstrap script.** `infra/vps-bootstrap.sh` (idempotent, run as root): `apt-get update && apt-get install -y docker.io docker-compose-v2 ufw unattended-upgrades`; create user `deploy` in group `docker` with your SSH key; `ufw default deny incoming; ufw allow 22,80,443/tcp; ufw --force enable`; `dpkg-reconfigure -f noninteractive unattended-upgrades`; `mkdir -p /opt/skillassert && chown deploy /opt/skillassert`. Run: `scp infra/vps-bootstrap.sh root@<ip>:/root/ && ssh root@<ip> bash /root/vps-bootstrap.sh`. Verify: `ssh deploy@<ip> docker ps` works; `ssh root@<ip> ufw status` lists only 22, 80, 443.
- [ ] **Step 4: First manual deploy.** `scp infra/docker-compose.prod.yml deploy@<ip>:/opt/skillassert/docker-compose.yml`; `scp infra/Caddyfile deploy@<ip>:/opt/skillassert/`; on the VPS create `/opt/skillassert/.env` with `DATABASE_URL=<session pooler URI>`, `APP_VERSION=manual`, mode 600. Push the image once by hand (`docker login ghcr.io`, tag, push), set the GHCR package to public in GitHub, then on the VPS `docker compose up -d`. Verify: `curl -s https://api.skillassert.com/health` returns `{"status":"ok","db":"ok","version":"manual"}` and `docker compose logs caddy | grep -i "certificate obtained"` prints a line.
- [ ] **Step 5: Review-focus checks.** On the VPS `ss -ltnp` lists only 22, 80 and 443 (the `docker-proxy` entries for 80 and 443 are Caddy's). Run `docker compose up -d --force-recreate` twice; `docker compose logs caddy` shows no second certificate issuance.
- [ ] **Step 6: Workflow job.** `deploy.yml` on `push` to `main`, job `api`: checkout; `docker/login-action@v3` to `ghcr.io` with `GITHUB_TOKEN`; `docker/build-push-action@v6` with tags `:latest` and `:${{ github.sha }}`; then over SSH (`appleboy/ssh-action@v1` or plain `ssh` with secrets `VPS_HOST`, `VPS_USER=deploy`, `VPS_SSH_KEY`): `cd /opt/skillassert && sed -i "s/^APP_VERSION=.*/APP_VERSION=${{ github.sha }}/" .env && docker compose pull && docker compose up -d && docker image prune -f`.
- [ ] **Step 7: Verify the roll-out.** Merge the pull request. Expected: the workflow succeeds and `curl -s https://api.skillassert.com/health` shows `"version":"<merge sha>"`.
- [ ] **Step 8: Commits.** Two commits on branch `build/api-container`: `build(api): add dockerfile, compose and caddy config` and `ci(deploy): build the api image and roll it out to the vps on main`.

---

### Task 8: Web deploy to Cloudflare Workers static assets with a client-shell fallback

**Files:**
- Create: `apps/web/wrangler.jsonc`, `apps/web/worker/index.ts`, `apps/web/worker/pick-fallback.ts`, `apps/web/worker/pick-fallback.spec.ts`
- Modify: `.github/workflows/deploy.yml` (job `web`), `package.json` (dev dep `wrangler`)

**Interfaces:**
- Produces: Worker `skillassert-web` with `assets: { directory: "../../dist/apps/web/browser", binding: "ASSETS", not_found_handling: "none" }`, routes `skillassert.com` and `www.skillassert.com` as custom domains; `pickFallback(pathname: string, accept: string | null): '/index.csr.html' | null` returning the shell for navigation requests (`accept` contains `text/html`) whose path has no file extension, else `null`; the fetch handler calls `env.ASSETS.fetch` with the fallback path or returns 404.

- [ ] **Step 1: Write the failing tests** in `pick-fallback.spec.ts`:

```ts
it('serves the client shell for a deep link', () => { expect(pickFallback('/quiz/start', 'text/html,*/*')).toBe('/index.csr.html'); });
it('does not serve the shell for a missing asset', () => { expect(pickFallback('/assets/missing.png', 'image/webp,*/*')).toBeNull(); });
it('does not serve the shell to non-html clients', () => { expect(pickFallback('/api/x', 'application/json')).toBeNull(); });
```

- [ ] **Step 2: Run them to verify they fail,** implement `pickFallback`, run again: PASS (`pnpm nx test web` picks the file up, or add it to the web project's test include).
- [ ] **Step 3: Worker and config.** `pnpm add -D wrangler`. `worker/index.ts`: default export with `fetch(request, env)` that computes `pickFallback(new URL(request.url).pathname, request.headers.get('accept'))` and returns `env.ASSETS.fetch(new Request(new URL(path, request.url), request))` or `new Response('Not found', { status: 404 })`. `wrangler.jsonc`: `name`, `main: "worker/index.ts"`, `compatibility_date` set to today, the `assets` block above, `routes: [{ pattern: "skillassert.com", custom_domain: true }, { pattern: "www.skillassert.com", custom_domain: true }]`.
- [ ] **Step 4: Deploy by hand once.** `pnpm nx build web --configuration=production`; `CLOUDFLARE_API_TOKEN=... CLOUDFLARE_ACCOUNT_ID=... pnpm wrangler deploy -c apps/web/wrangler.jsonc`. Expected: Wrangler prints the custom domains; Cloudflare creates their DNS records.
- [ ] **Step 5: Verify both behaviours.** `curl -s https://skillassert.com/ | grep -c Angular` prints 1 or more (prerendered); `curl -s -o /dev/null -w '%{http_code}' https://skillassert.com/quiz/start` prints 200 and `curl -s https://skillassert.com/quiz/start | grep -c '<sa-root'` prints 1 (client shell); `curl -s -o /dev/null -w '%{http_code}' https://skillassert.com/nope.png` prints 404.
- [ ] **Step 6: Workflow job.** `deploy.yml` job `web` (`needs: db`, see Task 9): checkout, node, `pnpm install --frozen-lockfile`, `pnpm nx build web --configuration=production`, `cloudflare/wrangler-action@v3` with `command: deploy -c apps/web/wrangler.jsonc` and secrets `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`.
- [ ] **Step 7: Commit.** Branch `ci/web-deploy`; `git commit -m "ci(deploy): deploy the web app to cloudflare workers static assets"`; pull request; after merge repeat Step 5.

---

### Task 9: Production migrations from CI

**Files:**
- Modify: `.github/workflows/deploy.yml` (job `db` first; `api` and `web` gain `needs: db`)

**Interfaces:**
- Consumes: secrets `SUPABASE_ACCESS_TOKEN`, `SUPABASE_PROJECT_REF`, `SUPABASE_DB_PASSWORD` from Task 0.

- [ ] **Step 1: Job.** `db`: checkout; `supabase/setup-cli@v1`; `supabase link --project-ref ${{ secrets.SUPABASE_PROJECT_REF }}` with `SUPABASE_ACCESS_TOKEN` in the environment; `supabase db push` with `SUPABASE_DB_PASSWORD` in the environment. Add `needs: db` to `api` and `web`.
- [ ] **Step 2: Verify the first run.** Merge the pull request (`ci(deploy): apply supabase migrations to production on main`). Expected: the `db` job applies two migrations; the production home page lists the launch tree.
- [ ] **Step 3: Verify idempotence.** Re-run the workflow from the Actions tab. Expected: `db` reports no new migrations and exits 0; `api` and `web` redeploy unchanged.

---

### Task 10: Repository README and the phase exit test

**Files:**
- Create: `README.md` (replace the generated one)

- [ ] **Step 1: README.** Prerequisites (Node, pnpm via corepack, Docker, Supabase CLI), local run (`pnpm install`, `supabase start`, `pnpm nx serve api`, `pnpm nx serve web`), tests (`pnpm nx run-many -t test`, `supabase test db`, `pnpm nx e2e web-e2e`), how deploys work (merge to `main`), and a pointer to `docs/planning/` for the design, map and plans.
- [ ] **Step 2: Run the Phase 0 exit test** from `docs/planning/plan.md` and paste the outputs into the pull request description.
- [ ] **Step 3: Commit.** `docs: add readme and record the phase 0 exit test`; pull request; merge. Update `plan.md` Phase 0 status to done with the date.
