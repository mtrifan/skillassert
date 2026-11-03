# skillassert.com planning workspace

Planning artifacts for skillassert.com, a quiz web app and personal learning project. Moved into the repository
on 2026-11-03 (Phase 0, Task 1 Step 9); GLOSSARY.md and the ADRs moved to the repo root, this folder stays the map.

## Layout

- `map.md`: the wayfinder map. Destination, Notes, Decisions so far, Not yet specified (the fog), Out of scope.
  Low-resolution view, loaded once per session.
- `issues/NN-slug.md`: one decision ticket per file, numbered from 01. Header lines: `Type:` (research | prototype |
  grilling | task), `Status:` (open | claimed | resolved), `Blocked by:` (ticket numbers). The answer goes under
  `## Answer` on resolution.
- `grilling-log.md`: the grilling rounds, questions with recommended answers and the answers given, appended per round.
- `research/`: findings written by research subagents, one cited file per research ticket.
- `/GLOSSARY.md` (repo root): domain language, created when the first term is resolved.
- `/docs/adr/` (repo root): architecture decision records, only for hard-to-reverse, surprising trade-offs.
- `design.md`: the approved v1 design (2026-10-29). `plan.md`: the phased roadmap with the learning syllabus.
  `plans/phase-N-*.md`: task-level plans, one per phase, written when the phase starts.

## How to start Phase 0 (and any later phase)

1. Do Task 0 of `plans/phase-0-walking-skeleton.md` yourself when convenient: GitHub repo, Cloudflare zone and the
   nameserver change at Porkbun (propagation can take a day, so do it early), Supabase project in Frankfurt, VPS,
   the `api` DNS record. No code, no session needed.
2. In a terminal run Task 1 Step 2 of the same plan from `~/Developer/Personal`; it creates `~/Developer/Personal/skillassert`.
3. Open a fresh agent session inside `~/Developer/Personal/skillassert` (Claude Code or Codex) and paste:

   > Read docs/planning/README.md, map.md, design.md, plan.md and plans/phase-0-walking-skeleton.md. We are
   > executing Phase 0 of skillassert.com step by step in this session, one task at a time, test first (Claude: use the superpowers
   > executing-plans skill). For every step tell me what we do, why and which concept it teaches, the exact
   > command or edit and who runs it, and the expected result; wait for my confirmation before ticking the
   > checkbox in the plan file and moving on. Secrets never go through the chat. Task 0 and Task 1 Step 2 are
   > done; start by verifying them, then continue from Task 1 Step 3.

4. For a later phase, first ask for the phase's task plan to be written from `plan.md` and `design.md` (Claude:
   superpowers writing-plans skill) into `docs/planning/plans/phase-N-<slug>.md`, commit it, then execute it the
   same way.

## Working from two machines (Claude on one, Codex on the other)

- The repository is the only shared state. After Task 1 Step 9 the planning workspace lives at `docs/planning/`
  in the repo and this folder on the Mac is a pointer. Claude Code's memory, Codex's memory and any `~/.claude`
  file are per machine and are not state.
- Both tools read `AGENTS.md` at the repo root (Claude through `CLAUDE.md` -> `@AGENTS.md`). Rules that must hold
  on both machines go there, nowhere else.
- Progress is the checkboxes in `docs/planning/plans/phase-N-*.md`, committed with the work they describe, plus
  one entry per session in `docs/planning/journal.md` (date, machine and tool, what was done, next step, gotchas).
- Protocol per session: `git pull --rebase` first; read the journal's last entry and the next unchecked step; one
  branch per task; never the same task from both machines at once; small pull requests; rebase, never merge
  commits; push before leaving the machine, even unfinished, as a draft pull request.
- Per machine, once: Node per `.nvmrc`, `corepack enable`, Docker, Supabase CLI, `pnpm install`, `pnpm exec
  playwright install`, `supabase start`, and a local `apps/api/.env` from `.env.example`. Local databases are
  rebuilt from migrations and seed, so they never need syncing. Production secrets live only in GitHub Actions and
  on the VPS.
- Windows machine: Docker Desktop with WSL2, work inside WSL, `.gitattributes` keeps line endings LF.
- Skills: superpowers and mattpocock-skills both ship Codex instructions; install them there for the same
  workflow, or rely on the plan files alone, which are explicit enough to execute without them.

## How to resume in a new session

1. Read `map.md` and this README.
2. Frontier = files in `issues/` with `Status: open` whose every `Blocked by:` ticket is `resolved`. First by number wins.
3. Claim it: set `Status: claimed` before any work.
4. Resolve it as its `Type:` says. grilling: grilling + domain-modeling skills, with the human. research: a subagent.
   prototype: the prototype skill. task: do it, or hand over a precise checklist.
5. Record it: `## Answer` in the ticket, `Status: resolved`, one gist line under "Decisions so far" in `map.md`.
   Graduate any fog the answer made specifiable into new tickets.
6. One ticket per session, except research tickets.

## Process reference

`/wayfinder` and `/grill-me` are not installed as slash commands in this Claude Code setup. The process is followed
by hand from the installed mattpocock-skills plugin:
`~/.claude/plugins/cache/claude-plugins-official/mattpocock-skills/<version>/skills/engineering/wayfinder/SKILL.md`,
together with `skills/productivity/grilling/SKILL.md` and `skills/engineering/domain-modeling/SKILL.md`.
Local-markdown tracker conventions: `skills/engineering/setup-matt-pocock-skills/issue-tracker-local.md`.

Hard gate (superpowers:brainstorming): nothing is built, scaffolded or installed until the design document is approved.
Prototypes and spikes are throwaway and say so.
