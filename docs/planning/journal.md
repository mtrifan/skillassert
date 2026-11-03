# Journal

One entry per session, newest last: date, machine and tool, what was done, next step, gotchas.

## 2026-11-03 · Mac · Claude Code (Opus 5.5), executing-plans, step by step with the owner

**Done**

- Task 0 Step 1: public repository `mtrifan/skillassert` created with `gh`. Task 0 Steps 2 to 7 are still open
  (owner, by hand). First task that needs them: Task 7.
- Task 1 complete, Steps 1 to 12. One commit `fed429e` on `main`, pushed. This was the only direct push to `main`.
- Task 2 Step 1 done on branch `chore/shared-domain` (draft pull request): zod 4.6.7, library `shared-domain`
  generated with tags `scope:shared,type:util`. Not ticked yet: the owner confirms the tick next session.

**Next step**

- Confirm the Task 2 Step 1 tick and the `testMode` change below, then Task 2 Step 2: write `health.spec.ts`,
  delete the generated placeholder `shared-domain.ts` and its spec, empty `index.ts`.

**Open decision**

- The `@nx/vitest` plugin in `nx.json` was registered with `"testMode": "watch"`, so `pnpm nx test shared-domain`
  stays in watch mode in a terminal and `run-many -t test` can hang locally. Proposal: set it to `"run"`
  (consistent with `web`'s `watch: false`); use `--watch` when an interactive loop is wanted.

**Rulings (deviations from the plan, each with its cost if wrong)**

- Task 1 Step 2: `--unitTestRunner=vitest` is invalid in Nx 24 (`vitest-angular | vitest-analog | jest | none`),
  so the generator set up no runner. Wired `vitest-angular` by hand as Nx's own generator does
  (`@angular/build:unit-test`, `tsconfig.spec.json`, vitest 4 and jsdom 27) and fixed the `nx.json` defaults.
  Cost: re-run the workspace generator.
- Task 1 Step 4: Angular 22 is zoneless by default and omits the provider; `provideZonelessChangeDetection()` added
  anyway to make the choice explicit. Cost: one redundant line.
- Task 1 Step 7: the generated e2e test asserted the welcome `h1`; it now asserts the shell header (the file must
  stay for Task 2 Step 7). Cost: one assertion.
- Task 1 Step 9: every employer reference removed from the planning docs before publishing (map, design, grilling
  log); the path-rewrite `sed` also hit Step 9's own text and stale README sentences, both restored by hand.
- Task 1 Step 11: `end_of_line = lf` added to `.editorconfig`.
- Task 2: branch created at Step 1 instead of Step 8; `--name=shared-domain` added so the plan's
  `nx test shared-domain` works (Nx 24 would name it `domain`).
- Task 8 (ahead): keep the plan's fallback Worker with `not_found_handling: "none"` over design section 13's
  `single-page-application`, which would serve the prerendered home for deep links.
- Task 8 (ahead): ship job `web` without `needs: db`; Task 9 adds it when job `db` exists.
- Task 6 (ahead, undecided): the Playwright config runs chromium, firefox and webkit, but CI installs only
  chromium. Recommendation: trim the config to chromium.
- Task 5 (ahead, undecided): with `outputMode: "static"`, `apps/web/src/server.ts` and `express` become unused.
  Recommendation: delete them then.

**Gotchas**

- Node: `.nvmrc` is 24; mise honours it after `mise settings add idiomatic_version_file_enable_tools node`.
  Corepack shims live per Node install, so `corepack enable` again after installing a new Node.
- Agent shells do not run mise's prompt hook and keep the old Node on PATH: prefix commands with `mise exec --`.
- Playwright pins browser builds to its version: after an upgrade run `pnpm exec playwright install`.
- zsh does not word-split `$var`; loops over "cmd arg" strings fail with "Both project and target have to be
  specified".
- Git identity on this Mac: everything under `~/Developer/Personal/` uses `~/Developer/Personal/.gitconfig` (noreply
  email, personal SSH key with `ssh -F /dev/null` so `~/.ssh/config`'s work key is never offered). `gh` does not
  switch by folder: `gh auth switch` to `mtrifan` before opening pull requests here.
- The executing-plans ledger at `.superpowers/` is git-ignored and local to this Mac; the rulings above are the
  durable copy.
