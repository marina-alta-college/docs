# Agentic Workflow Design — Marina Alta College

**Date:** 2026-05-05
**Status:** Approved (brainstorm)
**Next step:** Implementation plan via `superpowers:writing-plans`

## Overview

Set up `marina-alta-college` to run autonomous agent teams that take a client walkthrough transcript, produce specs, and ship merged PRs to a live Go + HTMX + Sanity website without human-in-the-loop on individual implementations. The pipeline mirrors `invoiceref`'s mattpocock-skills + slash-command + agent-team approach but is reshaped for a static marketing site (one builder agent instead of split backend/frontend, Playwright instead of iOS sim, full auto from spec → merged PR).

## Goals

1. Drop a meeting transcript in, get a triaged backlog of specs out.
2. Run `/implement <slug>` and walk away — agents commit to a branch, open a PR, CI gates auto-merge.
3. Sanity schema, Go templates, and i18n locale files stay structurally consistent (impossible to merge a mismatch).
4. No JavaScript ever ships in templates beyond the HTMX CDN script.
5. Design output is distinctive, not generic — bold-by-default exploration before each new section.

## Non-goals

- Mobile app development (out of scope for this project).
- Hand-tested visual regression baselines for every screen — every design change is intentional.
- HITL approval at the implementation stage.

## Architecture

### Repo consolidation

Today there are three independent GitHub repos under the `marina-alta-college` org: `site`, `studio`, `mood-board`. Target is one monorepo at `marina-alta-college/marina-alta-college` containing:

```
marina-alta-college/                    # ← becomes a git repo
├── .claude/                            # commands, agents, hooks, settings, team-templates
├── .agents/skills/                     # mattpocock skills (managed via skills-lock.json)
├── .github/workflows/                  # ci.yml, post-deploy.yml
├── docs/
│   ├── transcripts/                    # client walkthrough transcripts (committed)
│   ├── specs/                          # one spec per feature (output of /walkthrough)
│   ├── design/snapshots/               # Playwright baselines for visual checks
│   ├── design/explorations/            # design-explorer mockups, kept for reference
│   ├── agents/CONTEXT.md               # domain glossary for mattpocock skills
│   ├── adr/                            # architectural decisions
│   └── agent-log.md                    # per-task decisions, handoff notes
├── scripts/                            # check-no-js.sh, check-schema-sync.go, check-i18n.go
├── site/                               # Go SSR app (subtree-imported from existing repo)
├── studio/                             # Sanity Studio (subtree-imported)
├── Makefile                            # `make check` runs the full local gate
├── CLAUDE.md                           # top-level orchestration context
└── .gitignore
```

`mood-board/` stays as its own repo. It is a one-off design exploration sibling, not part of production.

### Migration approach

Use `git subtree add --prefix=site` and `--prefix=studio` to import each existing repo's full history into the new monorepo. Old `site` and `studio` GitHub repos get archived (not deleted) for historical reference. Render's connected repo updates to the new monorepo, root `site/`. Sanity studio deploy moves from manual `npm run deploy` to a CI job triggered on `studio/**` path changes.

## Pipeline

```
docs/transcripts/<file>.md  ──[/walkthrough]──>  docs/specs/<slug>.md  ──[auto-triage]──>
                                                                              │
                                                  ┌───────────────────────────┼───────────┐
                                                  ▼                           ▼           ▼
                                            ready-for-agent              needs-info    wontfix
                                                  │
                                       ──[you run /implement <slug>]──>
                                                  │
                                          [kind:new-section]?
                                                  │
                                       yes ──> design-explorer ──> chosen direction
                                                  │
                                                  ▼
                                              builder
                                                  │
                                                  ▼
                                         design-critic loop (max 3×)
                                                  │
                                                  ▼
                                            qa-reviewer
                                                  │
                                                  ▼
                                       opens PR ──> CI gate ──> auto-merge ──> Render + Sanity deploy ──> smoke check ──> agent-log entry
```

### Slash commands

| Command | Purpose |
|---|---|
| `/walkthrough [path]` | Read transcript, produce one spec per distinct feature mentioned, file GH issues with `needs-triage`, run auto-triage. |
| `/triage` | mattpocock skill — manage the issue backlog, transitions between states. |
| `/spec <slug>` | Hand-author a spec interactively (when there is no transcript). |
| `/implement <slug>` | Kick off agent team for one spec. Fully autonomous through merged PR. |
| `/log-decision [note]` | Append a decision block to `docs/agent-log.md`. |
| `/setup-matt-pocock-skills` | One-time bootstrap; sets up `docs/agents/CONTEXT.md`, label vocabulary, tracker config. |

### Auto-triage rules (initial)

The `/walkthrough` command runs auto-triage on every spec it produces, applying these rules:

- Spec touches **homepage hero or visual identity** → `kind:new-section`, state `ready-for-agent`.
- Spec adds **a new page or section** → `kind:new-section`, state `ready-for-agent`.
- Spec is a **copy change, typo fix, or content tweak** → `kind:tweak`, state `ready-for-agent`.
- Spec mentions **fees, scholarships, admissions deadlines, or any legally-binding figures** → state `needs-info` (you must confirm numbers before agent runs).
- Spec is **ambiguous or missing acceptance criteria** → state `needs-info`.
- Anything else → `kind:tweak`, state `ready-for-agent`.

The `kind` tag is set as a GitHub label and read by `/implement` to decide whether to spawn the design-explorer.

## Agent team

The team for `/implement` has three required roles plus one conditional:

| Agent | Role | When |
|---|---|---|
| `design-explorer` | Web research current design patterns (boutique schools, editorial, Mediterranean lifestyle), produce 2-3 distinct static HTML mockup directions using `frontend-design:frontend-design`, pick the bravest that fits brief. Output: chosen direction in `docs/design/explorations/<slug>/`. | Conditional — only when triage tagged `kind:new-section`. |
| `builder` | Owns the whole vertical: Sanity schema (in `studio/schemaTypes/`), Sanity content seed if needed, Go handlers + models in `site/`, html/template files, Tailwind/DaisyUI CSS, locale JSON for all 4 languages. End-to-end so it holds the schema↔template↔i18n triangle in one head. | Always. |
| `design-critic` | Starts local Go server, drives Playwright MCP through every changed route at desktop (1440px) + mobile (390px) breakpoints, captures screenshots, runs `design:design-critique` + `design:accessibility-review` + `design:ux-copy`. Returns structured fix list. PASS or BLOCK. Loops max 3× before escalating. | Always. |
| `qa-reviewer` | Final gate. Re-runs the local validation gate (build, no-JS check, schema sync, i18n completeness), confirms design-critic PASS, opens PR with structured body (spec link, screenshots, agent-log diff), enables auto-merge. | Always. |

### "Brave by default" prompts

- `design-explorer` prompt explicitly instructs: *"Default toward distinctive over safe. The client and Joe are not designers. Pull from current best-in-class boutique school sites, Editorial Luxe magazines, and Mediterranean lifestyle brands. If you find yourself producing the third generic Tailwind landing page, stop and start over."* WebSearch + WebFetch are first-class tools for this agent.
- `design-critic` prompt: *"You enforce the spec, but where the spec is silent on visual choices, prefer bolder over safer. Reject 'fine' — flag anything that looks AI-generic."*

## Validation gate

Two gates run the same checks: the local gate is the agent's pre-flight before opening the PR; the CI gate is the safety net before merge.

### Mandatory checks

| Check | Tool | Failure |
|---|---|---|
| Go build | `cd site && go build ./...` | Block |
| Go vet | `cd site && go vet ./...` | Block |
| Sanity build | `cd studio && npm run build` | Block |
| **No-JS guard** | `scripts/check-no-js.sh`: greps `site/templates/**` and `site/static/**` for `<script` tags, allowlist at `scripts/no-js-allowlist.txt` | Block |
| **Schema-template sync** | `scripts/check-schema-sync.go`: parses `studio/schemaTypes/*.ts` for field names, parses `site/sanity.go` GROQ queries + templates for `.FieldName` references, fails on mismatch | Block |
| **i18n completeness** | `scripts/check-i18n.go`: diffs key sets across `site/locales/{en,es,de,nl}.json`, fails on missing keys | Block |
| Playwright visual + axe | Local Go server up; Playwright MCP renders affected routes at 1440px + 390px; axe-core a11y scan; PNGs saved to `docs/design/snapshots/<slug>/` | Reviewed by design-critic |

### Soft check (warning only)

- Lighthouse CI against local server, perf budget 90 / a11y 95.

### CI workflow

`.github/workflows/ci.yml`:

- Triggers on every PR to main.
- Path filter: only `site/**` changed → skip Sanity build; only `studio/**` changed → skip Go build.
- Runs all mandatory checks.
- Auto-merge: PR labeled `automerge` (added by qa-reviewer when opening) → GitHub auto-merge fires when all required checks pass.
- Branch protection on `main`: required check `ci`, no human review required, auto-merge enabled.

### Post-deploy workflow

`.github/workflows/post-deploy.yml`:

- Triggers on push to main.
- Render auto-builds + deploys site (existing behavior, no GH Action needed).
- New job: runs `cd studio && npm run schema:deploy && npm run deploy` if `studio/**` changed in the merge commit. Requires `SANITY_AUTH_TOKEN` GitHub Secret.
- Smoke check: `curl https://marinaaltacollege.com` returns 200, body contains expected canary string. Failure prints to GH Action log (no notification — see "Notifications" below).

## `.claude/` layout

```
.claude/
├── settings.json              # hooks + permissions
├── settings.local.json        # gitignored, local overrides
├── repos.json                 # GH org/repo + deploy URLs
├── commands/
│   ├── walkthrough.md
│   ├── spec.md
│   ├── implement.md
│   └── log-decision.md
├── agents/
│   ├── design-explorer.md
│   ├── builder.md
│   ├── design-critic.md
│   └── qa-reviewer.md
├── team-templates/
│   └── implement-team.md      # instantiated per /implement run
├── hooks/
│   └── post-commit.sh         # appends one-line entry to docs/agent-log.md
└── skills -> ../.agents/skills  # symlink — mattpocock skills lock-managed
```

### Hooks

| Hook | What it does |
|---|---|
| `PostToolUse` Bash matching `git commit*` | Reads the just-made commit, appends one-line entry to `docs/agent-log.md` (commit sha, subject, slug detected from branch name). |
| `PreToolUse` Bash matching `git push*` to `origin main` | Refuses direct pushes to main; only allows pushing to `feat/**` branches. Belt-and-braces against an agent skipping the PR step. Bypassable only by explicit `--allow-main-push` env var. |

### Checkpointing

Every phase boundary in `/implement` writes `~/.claude/state/marina-alta/<slug>.checkpoint.json` with phase, last commit sha, validation state, outstanding todos, resume prompt. Lets you ctrl-C an `/implement` run and resume cleanly with `/resume <slug>`.

### Notifications

In-conversation text only. Each phase boundary in `/implement` prints one line to the conversation. No macOS notifications, no webhooks, no Discord. Joe is the only developer; the conversation is the channel.

### `repos.json`

```json
{
  "primary": {
    "github": "marina-alta-college/marina-alta-college",
    "default_branch": "main",
    "deploy_url": "https://marinaaltacollege.com",
    "studio_url": "https://marina-alta-college.sanity.studio"
  }
}
```

## Migration plan (implementation order)

### Phase A — Make the parent dir a repo + consolidate

1. `git init` in `marina-alta-college`, add origin remote.
2. Write root `.gitignore` covering `node_modules/`, `dist/`, `*.png` at root, `.DS_Store`, `~/.claude/state/`, `mood-board/` (kept as separate repo).
3. `git subtree add --prefix=site git@github.com:marina-alta-college/site.git main` and the same for `studio`. Preserves both histories.
4. Existing root `CLAUDE.md` stays in place as the orchestration context.
5. Initial push to `marina-alta-college/marina-alta-college`. Archive (do not delete) the old `site` and `studio` GitHub repos.
6. Update Render's connected repo to the new monorepo, root path `site/`. Verify build still works against the new path.

### Phase B — Scaffold `.claude/` + skills

1. Install mattpocock skills via `skills-lock.json`. Lock to the same shas invoiceref currently uses for repeatability.
2. Run `/setup-matt-pocock-skills` — writes `docs/agents/CONTEXT.md` (domain glossary), `docs/agents/issue-tracker.md` (GitHub).
3. Write the four agent definitions in `.claude/agents/`.
4. Write the team template in `.claude/team-templates/implement-team.md`.
5. Write the four slash commands in `.claude/commands/`.
6. Write `.claude/hooks/post-commit.sh` and the pre-push no-main-push guard.
7. Write `.claude/repos.json`, `.claude/settings.json` (hooks + permissions).

### Phase C — Validation scripts + CI

1. Write `scripts/check-no-js.sh`, `scripts/check-schema-sync.go`, `scripts/check-i18n.go`.
2. Add top-level `Makefile` with `make check` running all three plus `go build`, `go vet`, `npm run build` in studio.
3. Write `.github/workflows/ci.yml` (PR gate, paths-filter, auto-merge enabler).
4. Write `.github/workflows/post-deploy.yml` (Sanity studio deploy on `studio/**` change to main + smoke check).
5. Configure branch protection on `main` via `gh api`: required check `ci`, no human review required, auto-merge enabled.
6. Drop `SANITY_AUTH_TOKEN` into GitHub Secrets for the studio deploy job.

### Phase D — End-to-end dry run

1. Hand-write a tiny throwaway spec (e.g. "add a `tagline` field to siteSettings showing on home hero") and run `/implement` against it.
2. Verify the full chain: builder → critic → qa → PR → CI green → auto-merge → Render deploy → studio deploy → smoke check.
3. If anything breaks, fix it. The dry-run is the acceptance test for the whole installation.

## Day-to-day usage

1. Drop a transcript into `docs/transcripts/YYYY-MM-DD-<topic>.md`.
2. Run `/walkthrough <file>`. A backlog of triaged specs lands in `docs/specs/` + GH Issues.
3. Glance at the backlog, run `/implement <slug>` for one you want to ship.
4. Wait 10-30 min. A merged PR appears, the live site updates, `agent-log.md` has a new entry.
5. For ad-hoc changes without a transcript, use `/spec <slug>` (interactive) then `/implement <slug>`.

## Open questions for the implementation plan

- Exact subtree merge command sequence (the GitHub `site` and `studio` histories may have already-archived branches that need pruning before subtree add).
- Whether the no-JS allowlist needs entries for: future Plausible/analytics snippet, structured-data JSON-LD blocks (note: JSON-LD is `<script type="application/ld+json">` — the no-JS check needs to allow this since it is data, not behaviour).
- Whether `/implement` should be allowed to spawn parallel worktrees for multiple specs at once, or strictly serialize (recommend: serialize for v1; revisit if backlog volume justifies parallelism).

## References

- `/Users/joe/git/invoiceref/` — reference setup (HITL pipeline, mobile-app focused)
- `/Users/joe/git/invoiceref/.claude/commands/invoiceref-prd.md` — pattern for command structure
- `/Users/joe/git/invoiceref/.claude/agents/design-critic.md` — pattern for design-critic role
- `/Users/joe/git/invoiceref/skills-lock.json` — mattpocock skills lockfile shape
