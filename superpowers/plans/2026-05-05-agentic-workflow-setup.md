# Agentic Workflow Setup — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bootstrap `marina-alta-college` as a monorepo with autonomous agent infrastructure that takes client transcripts → triaged specs → fully-automated PRs merging to main, with Sanity-first invariants and no-JS guarantees enforced by CI.

**Architecture:** A single git repo at `/Users/joe/git/marina-alta-college` consolidates `site/` (Go SSR) and `studio/` (Sanity) via subtree merge. `.claude/` provides slash commands, agent definitions, and team templates that drive a `design-explorer → builder → design-critic → qa-reviewer` pipeline per spec, gated by `scripts/check-*.{sh,go}` validation locally and `.github/workflows/ci.yml` in CI. Auto-merge handles main updates; Render + a new GH Action handle deploys.

**Tech Stack:** Go 1.22+, Bash, GitHub Actions, Playwright MCP, Sanity v3, Anthropic design plugin skills, mattpocock skills (lock-managed via `skills-lock.json`).

**Spec:** `docs/superpowers/specs/2026-05-05-agentic-workflow-design.md`

---

## File Structure

### New files
```
.gitignore                                         (root)
.gitattributes                                     (root)
README.md                                          (root, brief monorepo overview)
Makefile                                           (root, `make check` aggregator)
skills-lock.json                                   (mattpocock skills lockfile)
.claude/settings.json
.claude/repos.json
.claude/commands/walkthrough.md
.claude/commands/spec.md
.claude/commands/implement.md
.claude/commands/log-decision.md
.claude/agents/design-explorer.md
.claude/agents/builder.md
.claude/agents/design-critic.md
.claude/agents/qa-reviewer.md
.claude/team-templates/implement-team.md
.claude/hooks/post-commit.sh
.claude/hooks/pre-push.sh
.claude/skills                                     (symlink → ../.agents/skills)
.agents/skills/                                    (mattpocock skills, lock-managed)
docs/agents/CONTEXT.md
docs/agents/issue-tracker.md
docs/agents/triage-labels.md
docs/agents/domain.md
docs/agent-log.md
docs/transcripts/.keep
docs/specs/.keep
docs/design/snapshots/.keep
docs/design/explorations/.keep
docs/adr/.keep
scripts/check-no-js.sh
scripts/no-js-allowlist.txt
scripts/check-schema-sync.go
scripts/check-schema-sync_test.go
scripts/check-i18n.go
scripts/check-i18n_test.go
scripts/go.mod
.github/workflows/ci.yml
.github/workflows/post-deploy.yml
```

### Files imported via subtree (preserved with full history)
```
site/**       (from github.com:marina-alta-college/site.git)
studio/**     (from github.com:marina-alta-college/studio.git)
```

### Files we leave untouched
```
CLAUDE.md                          (existing — top-level orchestration)
mood-board/                        (existing — separate repo, gitignored at root)
site/.claude/skills/site-change/   (existing — interactive HITL workflow, coexists with new /implement)
```

---

## Phase A — Repo Init + Consolidation

### Task A1: Pre-flight check on existing GitHub repos

**Files:** none (verification only)

- [ ] **Step 1: Verify both source repos are accessible from gh CLI**

```bash
gh repo view marina-alta-college/site --json url,defaultBranchRef
gh repo view marina-alta-college/studio --json url,defaultBranchRef
```
Expected: both return JSON with `defaultBranchRef.name == "main"`. If either is missing or named differently, stop and ask the user.

- [ ] **Step 2: Verify the destination repo does NOT yet exist**

```bash
gh repo view marina-alta-college/marina-alta-college 2>&1 | head -5
```
Expected: error "Could not resolve to a Repository". If it exists, stop and ask the user whether to use a different name or proceed (which would require a separate plan step to merge).

- [ ] **Step 3: Verify local working trees are clean**

```bash
cd /Users/joe/git/marina-alta-college/site && git status --short
cd /Users/joe/git/marina-alta-college/studio && git status --short
```
Expected: empty output for both (no uncommitted work to lose).

- [ ] **Step 4: Note current commit shas for the record**

```bash
cd /Users/joe/git/marina-alta-college/site && git rev-parse HEAD
cd /Users/joe/git/marina-alta-college/studio && git rev-parse HEAD
```
Record both shas in scratch notes — they are the consolidation provenance.

### Task A2: Create the destination GitHub repo

**Files:** none (GitHub-side action)

- [ ] **Step 1: Create the empty repo**

```bash
gh repo create marina-alta-college/marina-alta-college \
  --private \
  --description "Marina Alta College — Go + HTMX + Sanity monorepo" \
  --confirm
```
Expected: prints the new repo URL. (Use `--public` instead of `--private` if you want it public — confirm with user before running. Default to private.)

- [ ] **Step 2: Verify creation**

```bash
gh repo view marina-alta-college/marina-alta-college --json url,visibility
```
Expected: returns the URL and visibility. Note the URL.

### Task A3: Initialize the local monorepo

**Files:**
- Create: `/Users/joe/git/marina-alta-college/.gitignore`
- Create: `/Users/joe/git/marina-alta-college/.gitattributes`
- Create: `/Users/joe/git/marina-alta-college/README.md`

- [ ] **Step 1: `git init` at the parent dir**

```bash
cd /Users/joe/git/marina-alta-college
git init -b main
git remote add origin git@github.com:marina-alta-college/marina-alta-college.git
```
Expected: `Initialized empty Git repository`. `git remote -v` shows origin.

- [ ] **Step 2: Write the root `.gitignore`**

Path: `/Users/joe/git/marina-alta-college/.gitignore`

```gitignore
# OS
.DS_Store
*.swp

# Editor
.idea/
.vscode/

# Claude / agent state (per-machine)
.claude/settings.local.json
.agents/.cache/

# Node
node_modules/

# Go build output
site/site
site/mac-site
*.test
*.out

# CSS build output (generated)
site/static/css/output.css

# Sanity build output
studio/dist/
studio/.sanity/

# Sanity Studio per-user
studio/.env*

# Site env
site/.env

# Screenshots & dev artifacts at root (we keep ones under docs/design/)
/*.png
/*.jpeg
/*.jpg
.playwright-mcp/
lighthouse*.json

# Other repos that live as siblings inside this dir
mood-board/

# Logs
*.log
```

- [ ] **Step 3: Write `.gitattributes` for line endings**

Path: `/Users/joe/git/marina-alta-college/.gitattributes`

```
* text=auto eol=lf
*.png binary
*.jpg binary
*.jpeg binary
*.webp binary
*.pdf binary
```

- [ ] **Step 4: Write a minimal root `README.md`**

Path: `/Users/joe/git/marina-alta-college/README.md`

```markdown
# Marina Alta College

Monorepo for [Marina Alta College](https://marinaaltacollege.com).

## Layout

- `site/` — Go + HTMX production website. Deploys to Render on push to main.
- `studio/` — Sanity Studio. Deploys to `marina-alta-college.sanity.studio` via GitHub Actions.
- `docs/` — Specs, agent definitions, transcripts, design references.
- `.claude/` — Slash commands, agent definitions, team templates for autonomous development.
- `scripts/` — Validation scripts (no-JS guard, schema sync, i18n completeness).

## Development

```bash
cd site && make dev          # local dev server at :8080
cd studio && npm run dev     # local Sanity studio at :3333
make check                   # run the full local validation gate
```

## Agentic workflow

See `docs/superpowers/specs/2026-05-05-agentic-workflow-design.md`.

Quick path:

1. Drop a transcript in `docs/transcripts/`.
2. Run `/walkthrough <file>` — produces triaged specs.
3. Run `/implement <slug>` — agents take it from there.
```

- [ ] **Step 5: Initial commit**

```bash
cd /Users/joe/git/marina-alta-college
git add .gitignore .gitattributes README.md
git commit -m "init: marina alta college monorepo"
```
Expected: 3 files added. Note the commit sha.

### Task A4: Subtree-import `site/`

**Files:** modifies the working tree by importing `site/**` from the existing repo.

- [ ] **Step 1: Move the existing `site/` directory aside**

```bash
cd /Users/joe/git/marina-alta-college
mv site site.preimport.bak
```
Why: subtree merge needs an empty target path. We keep the old dir as a backup until we verify the import landed cleanly.
Expected: `ls` shows `site.preimport.bak/` instead of `site/`.

- [ ] **Step 2: Subtree-add the site repo**

```bash
cd /Users/joe/git/marina-alta-college
git subtree add --prefix=site git@github.com:marina-alta-college/site.git main
```
Expected: log lines about merging history, ending with a merge commit on main. `ls site/` shows the imported files.

- [ ] **Step 3: Verify the import matches the backup**

```bash
diff -rq /Users/joe/git/marina-alta-college/site /Users/joe/git/marina-alta-college/site.preimport.bak
```
Expected: only `Only in site.preimport.bak/.git: ...` differences (the original .git dir is not imported by subtree, which is what we want). Anything else is a divergence to investigate.

- [ ] **Step 4: Remove the backup**

```bash
rm -rf /Users/joe/git/marina-alta-college/site.preimport.bak
```
Expected: backup gone.

### Task A5: Subtree-import `studio/`

Same pattern as A4.

- [ ] **Step 1: Move existing `studio/` aside**

```bash
cd /Users/joe/git/marina-alta-college
mv studio studio.preimport.bak
```

- [ ] **Step 2: Subtree-add the studio repo**

```bash
git subtree add --prefix=studio git@github.com:marina-alta-college/studio.git main
```
Expected: merge commit on main. `ls studio/` shows imported files.

- [ ] **Step 3: Verify**

```bash
diff -rq /Users/joe/git/marina-alta-college/studio /Users/joe/git/marina-alta-college/studio.preimport.bak \
  | grep -v "Only in studio.preimport.bak/.git" \
  | grep -v "Only in studio.preimport.bak/node_modules"
```
Expected: empty output (modulo .git and node_modules).

- [ ] **Step 4: Remove backup**

```bash
rm -rf /Users/joe/git/marina-alta-college/studio.preimport.bak
```

### Task A6: Push initial monorepo to GitHub

**Files:** none (remote-side action)

- [ ] **Step 1: Push main to origin**

```bash
cd /Users/joe/git/marina-alta-college
git push -u origin main
```
Expected: push succeeds. `gh repo view marina-alta-college/marina-alta-college --json defaultBranchRef` returns `main`.

- [ ] **Step 2: Verify file presence on GitHub**

```bash
gh api repos/marina-alta-college/marina-alta-college/contents/site/main.go --jq .name
gh api repos/marina-alta-college/marina-alta-college/contents/studio/sanity.config.ts --jq .name
```
Expected: both return their filename.

### Task A7: Update Render to point at the new monorepo

**Files:** none (Render-side action; user-driven)

- [ ] **Step 1: Update Render's connected repo**

Action for the user (the agent should pause here and ask): in the Render dashboard for the existing `marinaaltacollege.com` service, switch the connected repo to `marina-alta-college/marina-alta-college` and set the root directory to `site/`. Build command and start command remain unchanged.

- [ ] **Step 2: Trigger a deploy and verify**

After the user confirms the dashboard change, push a no-op commit to confirm the new wiring:

```bash
cd /Users/joe/git/marina-alta-college
git commit --allow-empty -m "chore: trigger render redeploy from new monorepo"
git push origin main
```
Expected: Render dashboard shows a deploy from the new repo. After ~5 min, `curl -I https://marinaaltacollege.com` returns 200.

- [ ] **Step 3: Archive the old repos (do NOT delete)**

```bash
gh repo archive marina-alta-college/site --yes
gh repo archive marina-alta-college/studio --yes
```
Expected: both repos return `archived: true` from `gh repo view --json isArchived`.

---

## Phase B — Mattpocock skills + docs/agents/

### Task B1: Install mattpocock skills via lockfile

**Files:**
- Create: `/Users/joe/git/marina-alta-college/skills-lock.json`
- Create: `/Users/joe/git/marina-alta-college/.agents/skills/<each>/SKILL.md`
- Create: `/Users/joe/git/marina-alta-college/.claude/skills` (symlink)

- [ ] **Step 1: Copy the lockfile from invoiceref as a starting point**

```bash
cp /Users/joe/git/invoiceref/skills-lock.json /Users/joe/git/marina-alta-college/skills-lock.json
```
The invoiceref lockfile pins specific commit shas of `mattpocock/skills` for these skills: `diagnose`, `grill-me`, `grill-with-docs`, `improve-codebase-architecture`, `setup-matt-pocock-skills`, `to-issues`, `to-prd`, `triage`, `zoom-out`. We want all of them.

- [ ] **Step 2: Re-create the `.agents/skills/` tree by copying from invoiceref**

Since invoiceref already has these skills installed and we want the same versions:

```bash
mkdir -p /Users/joe/git/marina-alta-college/.agents
cp -R /Users/joe/git/invoiceref/.agents/skills /Users/joe/git/marina-alta-college/.agents/skills
ls /Users/joe/git/marina-alta-college/.agents/skills
```
Expected: `diagnose grill-me grill-with-docs improve-codebase-architecture setup-matt-pocock-skills to-issues to-prd triage zoom-out`.

- [ ] **Step 3: Create the `.claude/skills` symlink**

```bash
mkdir -p /Users/joe/git/marina-alta-college/.claude
cd /Users/joe/git/marina-alta-college/.claude
ln -s ../.agents/skills skills
ls -la skills
```
Expected: `skills -> ../.agents/skills`.

- [ ] **Step 4: Sanity-check the lock**

```bash
cd /Users/joe/git/marina-alta-college
cat skills-lock.json | jq '.skills | keys'
```
Expected: an array of 9 skill names matching the directory listing.

- [ ] **Step 5: Commit**

```bash
cd /Users/joe/git/marina-alta-college
git add skills-lock.json .agents/skills .claude/skills
git commit -m "chore: install mattpocock skills (locked from invoiceref)"
```

### Task B2: Write `docs/agents/CONTEXT.md` (domain glossary)

**Files:**
- Create: `/Users/joe/git/marina-alta-college/docs/agents/CONTEXT.md`

- [ ] **Step 1: Write the glossary**

Path: `/Users/joe/git/marina-alta-college/docs/agents/CONTEXT.md`

```markdown
# Marina Alta College — Domain Glossary

Authoritative vocabulary for the project. Agents read this before any non-trivial change. If a term is missing, add it here in the same PR that introduces it.

## Audience

- **Parent** — the decision-maker and budget-holder. Wealthy, liberal, entrepreneurial. Wants a bespoke education service. Not driven primarily by cost or by "brand British" prestige.
- **Prospective student** — age 16+, presented the school by their parent. May "flip" and present the school back to their parent once sold.
- **Visitor** — anyone on the public site, including curious community members and prospective hires.

## Programmes

- **A-level** — the academic programme offered. Two years (Year 12, Year 13). Exam-board agnostic in copy unless specified.
- **Subject** — an A-level offered (e.g. Mathematics, Biology). Subjects belong to a Programme.

## Admissions

- **Admissions** — the process by which a prospective student applies and is offered a place.
- **Fees** — annual tuition. Legally binding when published; treat any spec change touching fees as `needs-info` until confirmed.
- **Scholarship** — a fee reduction awarded on academic, artistic, sporting, or hardship grounds. Each scholarship has eligibility criteria + an award amount. Same `needs-info` rule as fees.
- **Open Day** — an event where prospective parents and students visit the school.

## Calendar

- **Term** — Autumn / Spring / Summer.
- **Half-term** — week-long break midway through a term.
- **Calendar event** — an entry on the public calendar. Types: `term`, `holiday`, `event`, `exam`.

## Content

- **Singleton** — a Sanity document type with exactly one instance (homePage, whoWeArePage, siteSettings). Document IDs are fixed (e.g. `homePage`).
- **Locale** — one of the four languages: `en`, `es`, `de`, `nl`. English is the source-of-truth and fallback.
- **Block** — a Portable Text content block in Sanity, rendered server-side by `site/render.go`.
- **CMS-editable** — content the client can change without a developer. All marketing copy, fees figures, staff bios, and journal entries must be CMS-editable. UI strings (button labels, error messages) are NOT CMS-editable; they live in `site/locales/*.json`.

## Site sections

- **Hero** — the first viewport-height section on a landing page.
- **Fast facts** — short factual claims displayed prominently (e.g. "Class size: 8").
- **Journal** — blog/news posts. Each is a `blogPost` document.
- **Founder's note** — singleton block on the Who We Are page; the founder's personal letter to readers.

## Tech vocabulary

- **GROQ** — Sanity's query language. All fetches live in `site/sanity.go`.
- **PageData** — the Go struct passed to every template (`site/render.go`). Contains `T` (translator), `Lang`, `Settings`, plus per-page data.
- **T helper** — Go template function `{{ T .T "key" }}` that looks up a locale string.
- **Stub** — fallback content used when Sanity is unavailable, defined per page in `site/handlers.go`.

## Workflow vocabulary

- **Spec** — markdown document at `docs/specs/<slug>.md` describing one feature.
- **Slug** — kebab-case identifier for a spec (e.g. `add-scholarships-page`).
- **Walkthrough** — a recorded conversation with the client whose transcript becomes specs.
- **Triage** — the state machine that moves an issue from `needs-triage` through to `ready-for-agent` / `needs-info` / `wontfix`.
- **Implement run** — a `/implement <slug>` invocation; one run produces one merged PR.
- **Validation gate** — the set of checks (build, no-JS, schema sync, i18n, design-critic, accessibility) that must pass before merge.
```

- [ ] **Step 2: Commit**

```bash
cd /Users/joe/git/marina-alta-college
git add docs/agents/CONTEXT.md
git commit -m "docs: add domain glossary for agent skills"
```

### Task B3: Write tracker + label config for mattpocock skills

**Files:**
- Create: `/Users/joe/git/marina-alta-college/docs/agents/issue-tracker.md`
- Create: `/Users/joe/git/marina-alta-college/docs/agents/triage-labels.md`
- Create: `/Users/joe/git/marina-alta-college/docs/agents/domain.md`

- [ ] **Step 1: Write `issue-tracker.md`**

Path: `/Users/joe/git/marina-alta-college/docs/agents/issue-tracker.md`

```markdown
# Issue tracker

Issues for this project live on **GitHub** at `marina-alta-college/marina-alta-college`. Skills that read or write issues use the `gh` CLI.

## Creating an issue

```bash
gh issue create \
  --repo marina-alta-college/marina-alta-college \
  --title "<title>" \
  --body "<body>" \
  --label "<labels>"
```

## Adding a label

```bash
gh issue edit <number> --repo marina-alta-college/marina-alta-college --add-label "<label>"
```

## Removing a label

```bash
gh issue edit <number> --repo marina-alta-college/marina-alta-college --remove-label "<label>"
```

## Listing issues by label

```bash
gh issue list --repo marina-alta-college/marina-alta-college --label "<label>" --state open
```

## Comments must include the AI disclaimer

Every triage comment posted by an agent **must** start with:
> *This was generated by AI during triage.*
```

- [ ] **Step 2: Write `triage-labels.md`**

Path: `/Users/joe/git/marina-alta-college/docs/agents/triage-labels.md`

```markdown
# Triage labels

Canonical role → label mapping for the mattpocock `triage` skill.

## Category labels

| Role | Label string |
|---|---|
| `bug` | `bug` |
| `enhancement` | `enhancement` |

## State labels

| Role | Label string |
|---|---|
| `needs-triage` | `needs-triage` |
| `needs-info` | `needs-info` |
| `ready-for-agent` | `ready-for-agent` |
| `ready-for-human` | `ready-for-human` |
| `wontfix` | `wontfix` |

## Marina-Alta-specific kind labels

The `/walkthrough` command also applies one of these to indicate which agent path the spec takes:

| Label | Meaning |
|---|---|
| `kind:new-section` | Net-new page or section. Triggers `design-explorer` before `builder`. |
| `kind:tweak` | Copy / minor visual / bug. Skips `design-explorer`. |

Every issue should carry exactly one category, one state, and (if `ready-for-agent`) one kind.
```

- [ ] **Step 3: Write `domain.md` (single-context layout)**

Path: `/Users/joe/git/marina-alta-college/docs/agents/domain.md`

```markdown
# Domain documentation layout

This repo is **single-context**: one `CONTEXT.md` and one `docs/adr/` for the whole monorepo.

- Domain glossary: `docs/agents/CONTEXT.md`
- Architectural decisions: `docs/adr/`
- Agent log (decisions, handoffs): `docs/agent-log.md`

The `site/` and `studio/` subdirs share this glossary. There is no separate frontend/backend context.
```

- [ ] **Step 4: Create the GitHub labels via gh API**

```bash
cd /Users/joe/git/marina-alta-college
for label in "needs-triage:fbca04" "needs-info:c5def5" "ready-for-agent:0e8a16" "ready-for-human:1d76db" "wontfix:cccccc" "kind:new-section:5319e7" "kind:tweak:bfd4f2"; do
  name="${label%:*}"
  color="${label##*:}"
  gh label create "$name" --color "$color" --repo marina-alta-college/marina-alta-college --force
done
```
Expected: each label creates or updates. (`bug` and `enhancement` already exist by default on a new GitHub repo, no need to create.)

- [ ] **Step 5: Commit**

```bash
cd /Users/joe/git/marina-alta-college
git add docs/agents/issue-tracker.md docs/agents/triage-labels.md docs/agents/domain.md
git commit -m "docs: configure mattpocock skill tracker + labels"
```

### Task B4: Create empty placeholders for transcripts/specs/design dirs

**Files:**
- Create: `docs/transcripts/.keep`
- Create: `docs/specs/.keep`
- Create: `docs/design/snapshots/.keep`
- Create: `docs/design/explorations/.keep`
- Create: `docs/adr/.keep`
- Create: `docs/agent-log.md` (empty seed)

- [ ] **Step 1: Create the empty dirs**

```bash
cd /Users/joe/git/marina-alta-college
mkdir -p docs/transcripts docs/specs docs/design/snapshots docs/design/explorations docs/adr
touch docs/transcripts/.keep docs/specs/.keep docs/design/snapshots/.keep docs/design/explorations/.keep docs/adr/.keep
```

- [ ] **Step 2: Seed `docs/agent-log.md`**

Path: `/Users/joe/git/marina-alta-college/docs/agent-log.md`

```markdown
# Agent log

Append-only log of decisions, handoffs, and notes from agent runs. New entries go at the **bottom** under the most recent date heading.

## 2026-05-05

- Monorepo bootstrapped from `site` + `studio` subtree merges. See `docs/superpowers/specs/2026-05-05-agentic-workflow-design.md`.
```

- [ ] **Step 3: Commit**

```bash
cd /Users/joe/git/marina-alta-college
git add docs/transcripts docs/specs docs/design docs/adr docs/agent-log.md
git commit -m "docs: seed working dirs (transcripts, specs, design, adr, agent log)"
```

---

## Phase C — Agent Definitions

### Task C1: Write `.claude/agents/builder.md`

**Files:**
- Create: `/Users/joe/git/marina-alta-college/.claude/agents/builder.md`

- [ ] **Step 1: Write the agent definition**

Path: `/Users/joe/git/marina-alta-college/.claude/agents/builder.md`

```markdown
---
name: builder
description: |
  Use this agent to implement the full vertical slice of a Marina Alta College spec: Sanity schema (in studio/schemaTypes/), Sanity content seed if needed, Go handlers + models in site/, html/template files, Tailwind/DaisyUI CSS, and locale JSON for all 4 languages. End-to-end so the schema↔template↔i18n triangle stays consistent.
model: inherit
---

You are the builder for Marina Alta College. You own one spec end-to-end.

## Inputs

- Spec at `docs/specs/<slug>.md` — read this first, fully.
- If `docs/design/explorations/<slug>/chosen.html` exists, that is the visual direction the design-explorer picked. Match it. The HTML may be different markup than what we ship (e.g. flatter than HTMX would emit) — capture the *visual intent*, not the literal markup.
- `docs/agents/CONTEXT.md` — domain glossary. Use these terms.
- `site/CLAUDE.md` and `studio/README.md` for stack-specific conventions.

## What you change

A typical slice touches all of:

1. **`studio/schemaTypes/<type>.ts`** — schema additions or new types. Every text field gets per-language variants (`en` required, `es` `de` `nl` optional with `en` fallback). Singletons use fixed document IDs (e.g. `homePage`, `siteSettings`).
2. **`site/models.go`** — Go struct mirroring the Sanity shape.
3. **`site/sanity.go`** — GROQ query string + fetch helper.
4. **`site/handlers.go`** — handler that fetches via Sanity helper, falls back to a stub when `SANITY_PROJECT_ID` is unset, populates `PageData`.
5. **`site/routes.go`** — register the route.
6. **`site/templates/pages/<page>.html`** or **`partials/<partial>.html`** — markup. SSR with `html/template`. HTMX attributes for any interactivity. Inside `{{range}}` loops, use `$` to access root `PageData` (e.g. `$.T`, `$.Settings`).
7. **`site/static/css/input.css`** — any custom CSS. After editing, rebuild: `npx @tailwindcss/cli -i static/css/input.css -o static/css/output.css --minify`.
8. **`site/locales/{en,es,de,nl}.json`** — every UI string used in the new template. **All four locale files must have the same keys**, even if values are duplicated until translations land.

## Hard rules

- **No JavaScript.** Do not write `<script>` tags except for the existing HTMX CDN script in `templates/base.html` and structured-data JSON-LD (`<script type="application/ld+json">`). The CI gate will reject any other script tag.
- **Sanity-first.** Never hardcode a piece of marketing copy in a Go template that the client might want to edit. If the spec touches written content, the schema gets the field, the handler reads it, the template renders it, the stub provides a fallback. The CI gate `check-schema-sync` rejects template references to fields that don't exist in schema.
- **i18n-complete.** Every locale JSON file must have every key. The CI gate `check-i18n` rejects missing keys.
- **No dark mode.** The site has one theme. If you find yourself adding `dark:` Tailwind classes, stop — you're solving the wrong problem.
- **Mobile-first, accessible.** Semantic HTML, proper heading hierarchy, ARIA labels on icon-only buttons, color contrast WCAG AA.
- **Stubs for offline.** The site renders without Sanity (`SANITY_PROJECT_ID` unset). New handlers must include a stub fallback so local development and design-critic Playwright runs don't depend on the live CMS.

## Workflow

1. Read the spec and the chosen design exploration (if any).
2. Plan the slice: list every file you will touch (the 8 categories above are a guide). If a category is N/A for this spec, say so explicitly.
3. Make schema changes in `studio/` first. Run `cd studio && npm run build` to verify.
4. Make Go changes. Run `cd site && go build ./...` and `go vet ./...` after each file.
5. Update locale JSONs in lockstep — never commit a locale change in isolation.
6. Rebuild CSS if you touched `input.css`.
7. Boot the local server (`cd site && make dev`) and hit your route in the browser to confirm it renders before handing off to design-critic.
8. Run `make check` (the local gate) at the repo root. Fix any failures before declaring done.
9. Hand off to `design-critic` with: route URL(s), Sanity changes summary, locale keys added.

## Sanity content updates (the critical invariant)

If the spec adds a new field or document type, you must also seed initial content so the field renders with something meaningful in dev. Use the Sanity CLI from `studio/`:

```bash
cd studio && npx sanity documents create '...'
```

Or if it's a singleton update, use `npx sanity documents replace --replace-existing`. Document the seed command in your handoff so the client can re-run it on prod if needed.

## What you do NOT do

- You do not commit. `qa-reviewer` opens the PR after the gate passes.
- You do not invent design choices when the spec is silent — escalate to `design-critic` (with PASS/BLOCK turnaround) or, if it's a major direction question, raise it as `needs-info` and stop.
- You do not skip the stub. Every new handler has a stub fallback.
```

- [ ] **Step 2: Commit**

```bash
cd /Users/joe/git/marina-alta-college
git add .claude/agents/builder.md
git commit -m "feat: add builder agent definition"
```

### Task C2: Write `.claude/agents/design-explorer.md`

**Files:**
- Create: `/Users/joe/git/marina-alta-college/.claude/agents/design-explorer.md`

- [ ] **Step 1: Write the agent definition**

Path: `/Users/joe/git/marina-alta-college/.claude/agents/design-explorer.md`

```markdown
---
name: design-explorer
description: |
  Use this agent BEFORE the builder, only when the spec is tagged kind:new-section. Researches current best-in-class design patterns on the web, generates 2-3 distinct visual directions as static HTML mockups via the frontend-design skill, and picks the bravest direction that fits the brief. Output goes to docs/design/explorations/<slug>/.
model: inherit
---

You are the design explorer for Marina Alta College. You are run BEFORE the builder for any spec tagged `kind:new-section`. Your job is to make sure what we ship is **distinctive**, not generic.

## Audience reminder (read before anything else)

The client is a wealthy, liberal, entrepreneurial parent looking for a *bespoke* education for their child. Not someone shopping on price. Not impressed by "brand British" prestige signalling. Wants candid, location-specific (Mediterranean, Moraira) imagery and a feeling of community and confidence.

The student is 16+, being shown the school by their parent. Should feel boutique and warm — not corporate, not chain-like, not childish.

## Inputs

- Spec at `docs/specs/<slug>.md`.
- `docs/agents/CONTEXT.md` — domain glossary.
- `site/CLAUDE.md` — current visual direction (Maritime Clean: cool blue-white, navy text, Mediterranean Aqua accent, Cormorant Garamond headings, DM Sans body).
- The existing site at `https://marinaaltacollege.com` (or local dev) — for visual continuity.

## Workflow

### 1. Research (WebSearch + WebFetch)

Spend a meaningful amount of context on this — research is the differentiator. Look at:

- **Best-in-class boutique school sites.** Search: "best boarding school websites 2026", "elite boutique school design", look at Le Rosey, Aiglon, Avenues, École Jeannine Manuel, lyceum-style sites.
- **Editorial / Editorial-Luxe magazine sites.** Search: "editorial magazine design 2026", look at Apartamento, Cabana Magazine, The Gentlewoman.
- **Mediterranean lifestyle brands.** Search: "Mediterranean lifestyle brand website", look at Sézane, La DoubleJ, Loro Piana's coastal collections.
- **Hospitality / boutique hotel sites near our location.** Specifically Costa Blanca + Moraira hotels.

For each reference you find compelling, capture:
- Screenshot description (visual layout, hierarchy, type pairings, palette)
- The one or two specific design moves that make it work
- Why it would or wouldn't work for our audience

### 2. Synthesize 2-3 distinct directions

Each direction should:
- Have a **name** that captures its character (e.g. "Editorial Almanac", "Coastal Bento", "Boutique Atlas").
- Have **one bold move** — something memorable that an AI-default Tailwind template wouldn't produce. A typographic system, an unusual layout grid, a content choreography choice, an unconventional use of negative space.
- **Reuse our existing palette and font stack** (Maritime Clean, Cormorant Garamond + DM Sans + Montserrat) — boldness comes from typography, layout, and editorial moves, not from changing the brand.

Generate each direction as a static HTML file (or a small set of HTML files for multi-screen specs) using the `frontend-design:frontend-design` skill. Save to:

```
docs/design/explorations/<slug>/
  ├── direction-1-<name>/
  │   └── index.html
  ├── direction-2-<name>/
  │   └── index.html
  ├── direction-3-<name>/        (optional)
  │   └── index.html
  └── README.md                  (one paragraph per direction + your pick)
```

### 3. Pick the bravest direction that fits the brief

In `README.md`, write:
- A one-paragraph summary of each direction.
- Your pick + why.
- A *brief intentions* section the builder can use: this list of design moves we want preserved when this becomes Go templates + HTMX + Tailwind.

Then write `docs/design/explorations/<slug>/chosen.html` as a copy of the chosen direction's main HTML — that's the file `builder` reads.

### 4. Hand off

Output to the orchestrator: the slug, the chosen direction name, and a one-sentence "the build target is …" handoff. The builder takes it from here.

## Tone instructions you should re-read every time

- **Default toward distinctive over safe.** The client and Joe are not designers. Pull from current best-in-class boutique school sites, editorial magazines, and Mediterranean lifestyle brands. If you find yourself producing the third generic Tailwind landing page, stop and start over.
- **Reject "fine".** If a direction is fine but forgettable, scrap it. Genuinely interesting > tasteful and forgotten.
- **Test against the audience, not against tradition.** "Boarding schools usually look like X" is irrelevant. We are not a boarding school chain. We are a 60-pupil college on the Mediterranean.

## What you do NOT do

- You do not change the brand palette or typeface stack. Boldness is in the layout/typography/editorial choices, not the colors.
- You do not implement in Go. Your output is HTML mockups; the builder translates them.
- You do not run unless triage tagged `kind:new-section`. For `kind:tweak` specs, the orchestrator skips you entirely.
```

- [ ] **Step 2: Commit**

```bash
cd /Users/joe/git/marina-alta-college
git add .claude/agents/design-explorer.md
git commit -m "feat: add design-explorer agent definition"
```

### Task C3: Write `.claude/agents/design-critic.md`

**Files:**
- Create: `/Users/joe/git/marina-alta-college/.claude/agents/design-critic.md`

- [ ] **Step 1: Write the agent definition**

Path: `/Users/joe/git/marina-alta-college/.claude/agents/design-critic.md`

```markdown
---
name: design-critic
description: |
  Use this agent after the builder finishes. Boots the local Go server, drives Playwright MCP through every changed route at desktop (1440px) and mobile (390px) breakpoints, captures screenshots, runs design:design-critique + design:accessibility-review + design:ux-copy. Returns structured fix list. PASS or BLOCK; max 3 critique loops before escalating.
model: inherit
---

You are the design critic for Marina Alta College. The user has stated visual design is a weak spot — your job is to make every screen visibly defensible against the brand and the spec before it ships.

## Inputs

- `builder` has just finished implementing.
- The spec at `docs/specs/<slug>.md` — specifically the "Acceptance criteria" and any "Visual notes" sections.
- The chosen design exploration at `docs/design/explorations/<slug>/chosen.html` if `kind:new-section`.
- Brand tokens: see `site/static/css/input.css` (Maritime Clean theme).

## Workflow

### 1. Boot the local stack

```bash
cd /Users/joe/git/marina-alta-college/site
make dev &  # CSS watcher + Go server on :8080
```

Confirm `curl -I http://localhost:8080/en` returns 200.

### 2. Drive Playwright MCP through every changed route

For each route in the spec's "Routes affected" list (or that you can infer from the diff):

- Capture at desktop (1440×900):
  ```
  mcp__playwright__browser_resize {"width": 1440, "height": 900}
  mcp__playwright__browser_navigate {"url": "http://localhost:8080/en/<route>"}
  mcp__playwright__browser_take_screenshot {"filename": "docs/design/snapshots/<slug>/<route>-desktop.png"}
  ```
- Capture at mobile (390×844):
  ```
  mcp__playwright__browser_resize {"width": 390, "height": 844}
  mcp__playwright__browser_take_screenshot {"filename": "docs/design/snapshots/<slug>/<route>-mobile.png"}
  ```
- For each interactive element (anchor, form, htmx-target), capture the hovered/active state if the visual delta is non-trivial.

`Read` each PNG path so the image enters your context.

### 3. Run a11y check via axe-core

```javascript
mcp__playwright__browser_evaluate {
  "function": "async () => { const r = await fetch('https://cdnjs.cloudflare.com/ajax/libs/axe-core/4.10.2/axe.min.js'); eval(await r.text()); const res = await axe.run(); return { violations: res.violations.length, details: res.violations.map(v => ({id: v.id, impact: v.impact, nodes: v.nodes.length})) }; }"
}
```

Any `critical` or `serious` violation is a Blocking item.

### 4. Run `design:design-critique` against each screenshot

For each route × breakpoint:
- Compare to the spec's expectations.
- Compare to the chosen exploration (if present) — call out any drift.
- Reject "AI-generic": stock card grids, generic gradient hero blobs, three-icon "feature row", anything that could be on any landing page.

### 5. Run `design:accessibility-review` if axe didn't already cover it

Color contrast, tap targets ≥44pt on mobile, focus-visible states, heading hierarchy, screen-reader-only text where icons appear without labels.

### 6. Run `design:ux-copy` on any new user-facing strings

Tone: confident, warm, specific. Plain English. No jargon. No emoji. No "lorem ipsum" leakage.

### 7. Output

```
## Design critique: <slug>

### Blocking (must fix before commit)
- <route>/<breakpoint>: <issue>. Fix: <specific change with file path>.

### Recommended (fix if quick)
- ...

### Suggestions (notes for future passes)
- ...

### PASS / BLOCK
- PASS  /  BLOCK <reason>
```

A single Blocking item BLOCKS. Hand back to `builder` with the fix list. After fixes, re-screenshot and re-critique. **Max 3 cycles**, then escalate to the user via the conversation channel and pause.

## Tone

- Where the spec is silent on visual choices, prefer **bolder over safer**. Reject "fine."
- The bar is "would this make a discerning prospective parent stop scrolling?", not "does this validate?"
- Don't pad the critique with platitudes. If it's good, say "PASS" and move on.

## What you do NOT do

- You don't write code. You hand a fix list back to `builder`.
- You don't redesign the brand. The palette and type stack are fixed.
- You don't approve based on a single golden-path screenshot. Empty / error / loading / mobile all matter.
```

- [ ] **Step 2: Commit**

```bash
cd /Users/joe/git/marina-alta-college
git add .claude/agents/design-critic.md
git commit -m "feat: add design-critic agent definition"
```

### Task C4: Write `.claude/agents/qa-reviewer.md`

**Files:**
- Create: `/Users/joe/git/marina-alta-college/.claude/agents/qa-reviewer.md`

- [ ] **Step 1: Write the agent definition**

Path: `/Users/joe/git/marina-alta-college/.claude/agents/qa-reviewer.md`

```markdown
---
name: qa-reviewer
description: |
  Use this agent as the final validation gate for any /implement run. Runs the full local validation gate (build, no-JS, schema sync, i18n, screenshots present, design-critic PASS), opens a PR with auto-merge enabled. Blocks if anything fails — no exceptions.
model: inherit
---

You are the QA reviewer and validation gate for Marina Alta College. You are the LAST step before a PR opens. Block on any failure.

## Inputs

- Spec at `docs/specs/<slug>.md`.
- `builder` has finished implementing.
- `design-critic` has returned a PASS (or escalated).
- Worktree: `/Users/joe/git/marina-alta-trees/<slug>` (created by `/implement`).
- Branch: `feat/<slug>`.

## Workflow

### 1. Run the full local validation gate

```bash
cd /Users/joe/git/marina-alta-college  # repo root
make check
```

`make check` runs (in this order):
- `cd site && go build ./...`
- `cd site && go vet ./...`
- `cd studio && npm run build`
- `scripts/check-no-js.sh`
- `cd scripts && go run check-schema-sync.go`
- `cd scripts && go run check-i18n.go`

Any non-zero exit BLOCKS. Hand back to `builder` with the failing output.

### 2. Confirm screenshots are present

For each route mentioned in the spec, both desktop (`<route>-desktop.png`) and mobile (`<route>-mobile.png`) PNGs exist under `docs/design/snapshots/<slug>/`. Missing screenshots BLOCK.

### 3. Confirm design-critic returned PASS

Look at the design-critic's most recent output for this slug in the conversation. If it's BLOCK or escalated, do not proceed.

### 4. Cross-check spec acceptance criteria

For every bullet in the spec's `## Acceptance criteria` section, verify the implementation covers it. Missing criteria BLOCK.

### 5. Stage and commit

```bash
cd <worktree>
git add -A
git commit -m "feat: <slug> <one-line summary>"
```

Subject is lowercase. The summary mirrors the spec's `## Overview` first sentence.

### 6. Push the branch

```bash
git push -u origin feat/<slug>
```

### 7. Open the PR with auto-merge

```bash
gh pr create \
  --repo marina-alta-college/marina-alta-college \
  --base main \
  --head feat/<slug> \
  --title "<slug>: <one-line summary>" \
  --body "$(cat <<'EOF'
## Summary

<2-3 bullet points from the spec overview>

## Spec
- `docs/specs/<slug>.md`

## Screenshots
<embedded references to docs/design/snapshots/<slug>/*.png>

## Validation
- [x] make check (build, no-JS, schema sync, i18n)
- [x] design-critic PASS
- [x] all spec acceptance criteria covered

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)" \
  --label "automerge"

gh pr merge --auto --squash --repo marina-alta-college/marina-alta-college <pr-number>
```

`--auto --squash` queues the merge to fire when CI is green.

### 8. Append to agent log

```bash
cd /Users/joe/git/marina-alta-college
cat >> docs/agent-log.md <<EOF

### <slug> — opened PR #<pr-number>
- Files: <list>
- Spec: \`docs/specs/<slug>.md\`
- PR: <pr-url>
- Date: $(date -u +%Y-%m-%dT%H:%M:%SZ)
EOF
git add docs/agent-log.md
git commit -m "docs: log PR for <slug>" -m "auto-merge will fire when CI passes"
git push origin feat/<slug>
```

### 9. Tell the conversation what happened

One line: `PR #<n> opened for <slug>. Auto-merge enabled. Watch <pr-url>.`

## Hard rules

- Never `--no-verify`. If a hook fails, fix the underlying issue.
- Never push to main directly. The pre-push hook will reject it.
- Never skip a check because "it'll probably be fine". The whole point of the gate is to catch the cases where it isn't.
- Never approve if `design-critic` BLOCKED. Hand back to `builder` instead.

## What you do NOT do

- You do not implement. That's `builder`.
- You do not critique design. That's `design-critic`.
- You do not write tests. The validation scripts ARE the tests.
```

- [ ] **Step 2: Commit**

```bash
cd /Users/joe/git/marina-alta-college
git add .claude/agents/qa-reviewer.md
git commit -m "feat: add qa-reviewer agent definition"
```

---

## Phase D — Slash Commands

### Task D1: Write `.claude/commands/walkthrough.md`

**Files:**
- Create: `/Users/joe/git/marina-alta-college/.claude/commands/walkthrough.md`

- [ ] **Step 1: Write the command**

Path: `/Users/joe/git/marina-alta-college/.claude/commands/walkthrough.md`

```markdown
---
description: "Read a client walkthrough transcript, produce one spec per distinct feature, file GitHub issues, and run auto-triage. Output: triaged backlog in docs/specs/ + GH Issues."
---

You are processing a client walkthrough transcript into specs.

Args: `$ARGUMENTS` — path to a transcript file under `docs/transcripts/`. If empty, use the most-recently-modified file in that directory.

## Phases

### 1. Read the transcript

If `$ARGUMENTS` is empty: `ls -t docs/transcripts/*.md | head -1` to find the most recent. Refuse if `docs/transcripts/` is empty.

Read the file fully. Skim once for structure (who's talking, what's discussed), then again to extract distinct features/changes.

### 2. Extract distinct features

A "feature" is anything the client (a) asked us to do, (b) flagged as missing, or (c) described as desired-future-state. Discussion of existing-and-fine is NOT a feature.

For each feature, capture:
- One-sentence description
- The client's stated motivation (the *why*)
- Whether it touches: copy, layout, new section, schema, fees/scholarships
- Direct quotes if helpful for spec authoring

Aim for *separable* features. If two things naturally ship together, it's one spec; if they could ship in different weeks, two.

### 3. Author one spec per feature

For each feature:

- Pick a kebab-case slug (e.g. `add-scholarships-page`).
- Refuse to overwrite an existing `docs/specs/<slug>.md` unless the user passed `--force`.
- Invoke the `to-prd` mattpocock skill, feeding it the conversation context for that feature. The skill uses the project's domain glossary (`docs/agents/CONTEXT.md`) and outputs a Marina-Alta-flavored spec.

Each spec must include:

```markdown
# <slug>: <one-line summary>

## Overview
2-4 sentences. What is being built and why.

## Source
- Transcript: `docs/transcripts/<filename>.md`
- Section/timestamp (if available): "<quote>"

## Acceptance criteria
Bulleted list. Each bullet is verifiable. The qa-reviewer will check every bullet.

## Routes affected
List the URLs the change touches, e.g. `/<lang>/scholarships`.

## Schema changes
Sanity schema additions or edits required (in `studio/schemaTypes/`). If none, write "None".

## Content updates
Sanity document content that needs seeding (e.g. populate the new singleton). If none, write "None".

## Visual notes
Any direction or guardrails on the visual. Empty if `kind:tweak`. May be filled by `design-explorer` for `kind:new-section`.

## Out of scope
Things mentioned in the transcript that are NOT in this spec. Critical for avoiding scope creep.
```

Save to `docs/specs/<slug>.md`.

### 4. File a GitHub issue per spec

```bash
gh issue create \
  --repo marina-alta-college/marina-alta-college \
  --title "<slug>: <one-line summary>" \
  --body "Spec: \`docs/specs/<slug>.md\`. Source: walkthrough <transcript-filename>." \
  --label "needs-triage,enhancement"
```

### 5. Run auto-triage on each issue

For each just-filed issue, apply the rules:

| Rule | Result |
|---|---|
| Mentions fees, scholarship amounts, admissions deadlines, or any binding figure | Add `needs-info` label. Add comment requesting confirmation. |
| Has clear acceptance criteria + no binding figures + adds new page/section | Add `kind:new-section`, `ready-for-agent`. Remove `needs-triage`. |
| Has clear acceptance criteria + no binding figures + only changes copy or fixes a bug | Add `kind:tweak`, `ready-for-agent`. Remove `needs-triage`. |
| Acceptance criteria are vague | Add `needs-info`. Comment what's unclear. |
| Otherwise (catch-all) | Leave as `needs-triage`. |

Every comment posted by this command starts with:
> *This was generated by AI during triage.*

### 6. Append to agent log

Open `docs/agent-log.md` and append (under today's date heading, creating it if missing):

```markdown
### <YYYY-MM-DD> — Walkthrough processed: <transcript-filename>
- Specs created: <slug-1>, <slug-2>, ...
- Triage outcomes: <count> ready-for-agent, <count> needs-info, <count> wontfix
```

### 7. Print summary to the conversation

One line per spec: slug, kind, state, GH issue URL. End with the next-step suggestion: `/triage` to review the backlog, or `/implement <slug>` to ship one.

## Conventions to encode

- Lowercase commit subjects.
- One spec ships one PR. Don't bundle.
- Anything mentioning fees / scholarship amounts is `needs-info` — Joe must confirm numbers.
- Don't try to be clever about transcript structure. If you can't extract a feature cleanly, list it as a question in `docs/agent-log.md` rather than authoring a half-spec.
```

- [ ] **Step 2: Commit**

```bash
cd /Users/joe/git/marina-alta-college
git add .claude/commands/walkthrough.md
git commit -m "feat: add /walkthrough command"
```

### Task D2: Write `.claude/commands/spec.md`

**Files:**
- Create: `/Users/joe/git/marina-alta-college/.claude/commands/spec.md`

- [ ] **Step 1: Write the command**

Path: `/Users/joe/git/marina-alta-college/.claude/commands/spec.md`

```markdown
---
description: "Hand-author a single spec interactively. Use when there is no transcript and you want one feature spec written via brainstorming + grill-with-docs."
---

You are authoring a spec for Marina Alta College, interactively, from the user's prompt.

Args: `$ARGUMENTS` — kebab-case slug. Required. Refuse if the slug is empty or already exists at `docs/specs/<slug>.md` (unless `--force`).

## Phases

### 1. Brainstorm

Invoke the `superpowers:brainstorming` skill. Elicit:
- Goal (one sentence)
- User benefit
- Acceptance criteria (specific and verifiable)
- Routes affected
- Schema changes
- Content seed needed
- Visual direction (or "open" if `kind:new-section` and design-explorer should decide)
- Out-of-scope items

### 2. Grill with docs

Invoke the `grill-with-docs` skill (or its equivalent inline behaviour: cross-reference the brainstorm against `docs/agents/CONTEXT.md`; surface any term the spec invents that the codebase already names differently).

### 3. Write the spec

Save to `docs/specs/<slug>.md` using the same template as `/walkthrough`.

### 4. File a GitHub issue

```bash
gh issue create \
  --repo marina-alta-college/marina-alta-college \
  --title "<slug>: <one-line summary>" \
  --body "Spec: \`docs/specs/<slug>.md\`. Authored by /spec (no transcript)." \
  --label "needs-triage,enhancement"
```

### 5. Apply triage labels

Same rules as `/walkthrough` step 5.

### 6. Append to agent log

```markdown
### <YYYY-MM-DD> — Spec authored: <slug>
- Source: /spec (no transcript)
- Triage state: <ready-for-agent | needs-info>
- Issue: <gh-url>
```

### 7. Print summary + next step

`Spec written to docs/specs/<slug>.md, issue <url>. Next: /implement <slug>` (or `/triage` if `needs-info`).
```

- [ ] **Step 2: Commit**

```bash
cd /Users/joe/git/marina-alta-college
git add .claude/commands/spec.md
git commit -m "feat: add /spec command"
```

### Task D3: Write `.claude/commands/implement.md`

**Files:**
- Create: `/Users/joe/git/marina-alta-college/.claude/commands/implement.md`

- [ ] **Step 1: Write the command**

Path: `/Users/joe/git/marina-alta-college/.claude/commands/implement.md`

```markdown
---
description: "Run the full autonomous pipeline for one spec: design-explorer (if kind:new-section) → builder → design-critic → qa-reviewer → PR with auto-merge. No HITL after kickoff."
---

You are orchestrating the autonomous pipeline for one Marina Alta College spec.

Args: `$ARGUMENTS` — slug. Required.

## Pre-flight

### 1. Verify the spec exists

`ls docs/specs/<slug>.md`. Stop if missing — tell the user to run `/spec <slug>` or `/walkthrough` first.

### 2. Read the spec + figure out the kind

Read `docs/specs/<slug>.md`. Look up the GitHub issue (`gh issue list --search "<slug>" --label kind:new-section --json number,labels`). The `kind:` label determines whether `design-explorer` runs.

If the issue is labeled `needs-info`, refuse and tell the user to resolve the open questions first.

### 3. Open a worktree

Use the `superpowers:using-git-worktrees` skill. Branch: `feat/<slug>`. Worktree path: `/Users/joe/git/marina-alta-trees/<slug>`.

### 4. Write the checkpoint

Path: `~/.claude/state/marina-alta/<slug>.checkpoint.json`

```json
{
  "slug": "<slug>",
  "spec_path": "docs/specs/<slug>.md",
  "kind": "<new-section | tweak>",
  "worktree": "/Users/joe/git/marina-alta-trees/<slug>",
  "branch": "feat/<slug>",
  "phase": "starting",
  "last_commit_sha": "<HEAD sha at worktree creation>",
  "validation_state": { "build": "pending", "no-js": "pending", "schema-sync": "pending", "i18n": "pending", "design-critic": "pending" },
  "updated_at": "<ISO 8601>"
}
```

`mkdir -p ~/.claude/state/marina-alta` if needed.

### 5. Print the kickoff line

`Implementing <slug> in worktree <path>. Kind: <kind>. Phases: explorer? → builder → critic → qa.`

## Pipeline

### Phase 1: design-explorer (only if kind:new-section)

Spawn the `design-explorer` agent via the `Agent` tool with `subagent_type: design-explorer` (or via raw Agent if subagent_type is unsupported, in which case pass the full content of `.claude/agents/design-explorer.md` as a system prompt). Pass:
- `slug`
- `spec_path`
- `worktree`

Wait for completion. The agent writes `docs/design/explorations/<slug>/chosen.html`. Update checkpoint phase to `explorer-done`.

### Phase 2: builder

Spawn the `builder` agent. Pass slug, spec, worktree, plus the explorer's chosen-direction summary (if any).

Wait for completion. Update checkpoint phase to `builder-done`.

### Phase 3: design-critic loop (max 3 iterations)

Spawn `design-critic`. If output is PASS → continue.

If output is BLOCK → spawn `builder` again with the critic's fix list. Re-run `design-critic`. Iterate up to 3 times total. After 3 BLOCKs, escalate: update checkpoint phase to `escalated`, print to conversation: `<slug> escalated after 3 critic BLOCKs. Pausing. Run /resume <slug> after addressing.` and stop.

### Phase 4: qa-reviewer

Spawn `qa-reviewer`. The agent runs `make check`, opens the PR with `--label automerge`, and enables auto-merge via `gh pr merge --auto --squash`.

Wait for the PR URL. Update checkpoint phase to `pr-opened`. Print: `PR <url> opened. Auto-merge will fire when CI passes.`

### Phase 5: monitor (light)

Optional: `gh pr checks <pr> --repo ...` once and report status. Then exit. The PR will merge itself when CI passes; the user does not need to wait inside this command.

## On any failure

If any agent crashes or returns an unrecoverable error:
- Update checkpoint to `phase: failed`, `error: <description>`.
- Print to conversation what happened.
- Do not auto-retry. The user runs `/resume <slug>` after fixing.

## Completion line

`Done with <slug>. PR: <url>. Watch for the merge.`
```

- [ ] **Step 2: Commit**

```bash
cd /Users/joe/git/marina-alta-college
git add .claude/commands/implement.md
git commit -m "feat: add /implement command"
```

### Task D4: Write `.claude/commands/log-decision.md`

**Files:**
- Create: `/Users/joe/git/marina-alta-college/.claude/commands/log-decision.md`

- [ ] **Step 1: Write the command**

Path: `/Users/joe/git/marina-alta-college/.claude/commands/log-decision.md`

```markdown
---
description: "Append a decision block to docs/agent-log.md. Use whenever you make a decision, change of plan, or hit something a future agent would need to know."
---

You are appending a decision to the agent log.

Args: `$ARGUMENTS` — the decision text. If empty, prompt the user for it.

## What to write

Append to `docs/agent-log.md` under today's date heading (`## YYYY-MM-DD`, creating it if missing):

```markdown
### <HH:MM> — <one-line summary>
<the decision text, expanded with context — what was decided, why, what alternatives were rejected>
```

Then commit:

```bash
git add docs/agent-log.md
git commit -m "docs: log decision — <one-line summary>"
```

## Tone

- Past-tense factual. "Decided to ship X because Y. Rejected Z because W."
- Include enough context that an agent picking this up next week understands without re-reading the conversation.
- If the decision affects an open spec, link the spec.
```

- [ ] **Step 2: Commit**

```bash
cd /Users/joe/git/marina-alta-college
git add .claude/commands/log-decision.md
git commit -m "feat: add /log-decision command"
```

---

## Phase E — Team Template + Hooks + Settings

### Task E1: Write `.claude/team-templates/implement-team.md`

**Files:**
- Create: `/Users/joe/git/marina-alta-college/.claude/team-templates/implement-team.md`

- [ ] **Step 1: Write the team template**

Path: `/Users/joe/git/marina-alta-college/.claude/team-templates/implement-team.md`

```markdown
# implement-team template

Used by `/implement <slug>`. The orchestrating session reads this and instantiates the team via the `TeamCreate` tool with placeholders filled in. (If `TeamCreate` is unavailable, the orchestrator dispatches each agent serially via the `Agent` tool, which is the primary path documented in `.claude/commands/implement.md`.)

## Team shape

| Member | Agent file | Model | When |
|---|---|---|---|
| `design-explorer` | `.claude/agents/design-explorer.md` | inherit | only if `kind:new-section` |
| `builder` | `.claude/agents/builder.md` | inherit | always |
| `design-critic` | `.claude/agents/design-critic.md` | inherit | always |
| `qa-reviewer` | `.claude/agents/qa-reviewer.md` | inherit | always |

All members share the same `cwd`: `{{WORKTREE}}`.

## Dispatch order

1. `design-explorer` (conditional) → output: `docs/design/explorations/{{SLUG}}/chosen.html` + summary.
2. `builder` → produces the working slice. Hands off route URLs to design-critic.
3. `design-critic` loop (max 3) → PASS or BLOCK. On BLOCK, builder re-runs with the fix list.
4. `qa-reviewer` → runs `make check`, opens PR with `automerge` label, enables auto-merge.

## Initial prompt template per role

```
You are the {{ROLE}} for /implement run on slug {{SLUG}}.

## Spec
- Path: {{SPEC_PATH}}
- Kind: {{KIND}}

## Workspace
- Worktree: {{WORKTREE}}
- Branch: feat/{{SLUG}}
- Checkpoint: ~/.claude/state/marina-alta/{{SLUG}}.checkpoint.json

## Your role
Read your agent definition at .claude/agents/{{ROLE}}.md before doing anything. Follow the dispatch order in .claude/team-templates/implement-team.md.

## Project conventions
- Lowercase commit subjects.
- No JS in templates.
- Sanity-first: schema gets the field BEFORE the template references it.
- All four locales updated in lockstep.
- Append decisions to docs/agent-log.md.
```

## Placeholders

| Placeholder | Source |
|---|---|
| `{{SLUG}}` | Arg to `/implement` |
| `{{SPEC_PATH}}` | `docs/specs/{{SLUG}}.md` |
| `{{KIND}}` | From the GH issue's `kind:*` label |
| `{{WORKTREE}}` | `/Users/joe/git/marina-alta-trees/{{SLUG}}` |
| `{{ROLE}}` | The agent's role name |
```

- [ ] **Step 2: Commit**

```bash
cd /Users/joe/git/marina-alta-college
git add .claude/team-templates/implement-team.md
git commit -m "feat: add implement-team template"
```

### Task E2: Write `.claude/hooks/post-commit.sh`

**Files:**
- Create: `/Users/joe/git/marina-alta-college/.claude/hooks/post-commit.sh`

- [ ] **Step 1: Write the hook**

Path: `/Users/joe/git/marina-alta-college/.claude/hooks/post-commit.sh`

```bash
#!/usr/bin/env bash
# Appends a one-line entry to docs/agent-log.md after each git commit made
# by the agent. Triggered by .claude/settings.json PostToolUse Bash matcher.
set -euo pipefail

# Find the repo root (the parent dir that holds .git).
repo_root=$(git rev-parse --show-toplevel 2>/dev/null || echo "")
if [[ -z "$repo_root" || ! -d "$repo_root/.git" ]]; then
  exit 0  # not in a repo, nothing to log
fi

# Skip if this isn't the marina-alta-college repo (safety net for dev-time misuse).
case "$repo_root" in
  */marina-alta-college) ;;
  *) exit 0 ;;
esac

log_path="$repo_root/docs/agent-log.md"
if [[ ! -f "$log_path" ]]; then
  exit 0  # log doesn't exist yet (e.g. first commit)
fi

# Get the latest commit info.
sha=$(git -C "$repo_root" rev-parse --short HEAD)
subject=$(git -C "$repo_root" log -1 --pretty=%s)

# Try to detect a slug from the current branch (feat/<slug>).
branch=$(git -C "$repo_root" rev-parse --abbrev-ref HEAD)
slug=""
case "$branch" in
  feat/*) slug="${branch#feat/}" ;;
esac

today=$(date -u +%Y-%m-%d)

# Ensure today's date heading exists in the log.
if ! grep -q "^## $today\$" "$log_path"; then
  printf '\n## %s\n' "$today" >> "$log_path"
fi

# Append the commit line.
if [[ -n "$slug" ]]; then
  printf -- '- %s [`%s`] (%s) — %s\n' "$sha" "$slug" "$branch" "$subject" >> "$log_path"
else
  printf -- '- %s — %s\n' "$sha" "$subject" >> "$log_path"
fi
```

- [ ] **Step 2: Make it executable**

```bash
chmod +x /Users/joe/git/marina-alta-college/.claude/hooks/post-commit.sh
```

- [ ] **Step 3: Smoke-test**

```bash
cd /tmp && bash /Users/joe/git/marina-alta-college/.claude/hooks/post-commit.sh
```
Expected: exits 0 silently (we're not in marina-alta-college repo). No error.

- [ ] **Step 4: Commit**

```bash
cd /Users/joe/git/marina-alta-college
git add .claude/hooks/post-commit.sh
git commit -m "feat: add post-commit hook for agent-log"
```

### Task E3: Write `.claude/hooks/pre-push.sh`

**Files:**
- Create: `/Users/joe/git/marina-alta-college/.claude/hooks/pre-push.sh`

- [ ] **Step 1: Write the hook**

Path: `/Users/joe/git/marina-alta-college/.claude/hooks/pre-push.sh`

```bash
#!/usr/bin/env bash
# Refuses pushes directly to origin/main. The auto-merge does main updates.
# Bypassable for emergencies via ALLOW_MAIN_PUSH=1 env var.
set -euo pipefail

# Read the input — git's pre-push hook gets refspecs on stdin.
# But we're called as a Claude Code hook on Bash(git push*), not as a real git
# pre-push hook. Inspect the command line of the parent process instead.
cmd="${CLAUDE_HOOK_BASH_COMMAND:-}"
if [[ -z "$cmd" ]]; then
  # Fall back: if we can't see the command, allow the push.
  exit 0
fi

# Allow if explicitly bypassed.
if [[ "${ALLOW_MAIN_PUSH:-0}" == "1" ]]; then
  exit 0
fi

# Block git push origin main / git push origin HEAD:main / git push --all etc.
case "$cmd" in
  *"git push"*"origin"*"main"*) ;;
  *"git push"*"--all"*) ;;
  *"git push"*"HEAD:main"*) ;;
  *) exit 0 ;;  # not pushing to main, allow
esac

cat >&2 <<EOF
Refusing direct push to main.

Marina Alta College is configured for PR-with-auto-merge. Pushing directly
to main bypasses the CI gate (no-JS check, schema-sync, i18n completeness,
design-critic).

If this is an emergency hotfix:
  ALLOW_MAIN_PUSH=1 <your push command>

Otherwise, push to a feat/<slug> branch and let qa-reviewer open the PR.
EOF
exit 1
```

- [ ] **Step 2: Make it executable**

```bash
chmod +x /Users/joe/git/marina-alta-college/.claude/hooks/pre-push.sh
```

- [ ] **Step 3: Smoke-test**

```bash
CLAUDE_HOOK_BASH_COMMAND='git push origin feat/x' bash /Users/joe/git/marina-alta-college/.claude/hooks/pre-push.sh; echo "exit=$?"
CLAUDE_HOOK_BASH_COMMAND='git push origin main' bash /Users/joe/git/marina-alta-college/.claude/hooks/pre-push.sh; echo "exit=$?"
ALLOW_MAIN_PUSH=1 CLAUDE_HOOK_BASH_COMMAND='git push origin main' bash /Users/joe/git/marina-alta-college/.claude/hooks/pre-push.sh; echo "exit=$?"
```
Expected: `exit=0`, `exit=1`, `exit=0`.

- [ ] **Step 4: Commit**

```bash
cd /Users/joe/git/marina-alta-college
git add .claude/hooks/pre-push.sh
git commit -m "feat: add pre-push hook to block direct main pushes"
```

### Task E4: Write `.claude/settings.json` and `.claude/repos.json`

**Files:**
- Create: `/Users/joe/git/marina-alta-college/.claude/settings.json`
- Create: `/Users/joe/git/marina-alta-college/.claude/repos.json`

- [ ] **Step 1: Write `settings.json`**

Path: `/Users/joe/git/marina-alta-college/.claude/settings.json`

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/Users/joe/git/marina-alta-college/.claude/hooks/post-commit.sh",
            "if": "Bash(git commit*)",
            "timeout": 10,
            "statusMessage": "Logging commit to agent-log"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/Users/joe/git/marina-alta-college/.claude/hooks/pre-push.sh",
            "if": "Bash(git push*)",
            "timeout": 5,
            "statusMessage": "Checking push target"
          }
        ]
      }
    ]
  }
}
```

- [ ] **Step 2: Write `repos.json`**

Path: `/Users/joe/git/marina-alta-college/.claude/repos.json`

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

- [ ] **Step 3: Commit**

```bash
cd /Users/joe/git/marina-alta-college
git add .claude/settings.json .claude/repos.json
git commit -m "feat: add .claude settings + repos config"
```

---

## Phase F — Validation Scripts (TDD)

### Task F1: Write `scripts/check-no-js.sh`

**Files:**
- Create: `/Users/joe/git/marina-alta-college/scripts/check-no-js.sh`
- Create: `/Users/joe/git/marina-alta-college/scripts/no-js-allowlist.txt`
- Create: `/Users/joe/git/marina-alta-college/scripts/check-no-js_test.sh`

- [ ] **Step 1: Write the test (failing) FIRST**

Path: `/Users/joe/git/marina-alta-college/scripts/check-no-js_test.sh`

```bash
#!/usr/bin/env bash
# Tests for check-no-js.sh.
set -euo pipefail

script="$(dirname "$0")/check-no-js.sh"
allowlist="$(dirname "$0")/no-js-allowlist.txt"
tmp=$(mktemp -d)
trap "rm -rf '$tmp'" EXIT

mkdir -p "$tmp/site/templates" "$tmp/site/static"

pass=0; fail=0

assert_pass() {
  local name="$1"; shift
  if "$script" "$tmp/site" "$allowlist" 2>/dev/null; then
    echo "PASS: $name"; pass=$((pass+1))
  else
    echo "FAIL: $name (expected exit 0)"; fail=$((fail+1))
  fi
}

assert_fail() {
  local name="$1"; shift
  if "$script" "$tmp/site" "$allowlist" 2>/dev/null; then
    echo "FAIL: $name (expected non-zero)"; fail=$((fail+1))
  else
    echo "PASS: $name"; pass=$((pass+1))
  fi
}

# Test 1: empty templates dir → pass
rm -rf "$tmp/site/templates"; mkdir "$tmp/site/templates"
assert_pass "empty templates"

# Test 2: HTMX CDN script (allowlisted) → pass
cat > "$tmp/site/templates/base.html" <<'HTML'
<!DOCTYPE html>
<html>
<head><script src="https://unpkg.com/htmx.org@2.0.4"></script></head>
<body></body>
</html>
HTML
assert_pass "htmx cdn allowlisted"

# Test 3: JSON-LD script (allowlisted) → pass
cat > "$tmp/site/templates/seo.html" <<'HTML'
<script type="application/ld+json">{"@context":"https://schema.org"}</script>
HTML
assert_pass "json-ld allowlisted"

# Test 4: arbitrary inline script → fail
cat > "$tmp/site/templates/bad.html" <<'HTML'
<script>console.log("hi");</script>
HTML
assert_fail "inline script blocked"

# Test 5: arbitrary external script → fail
rm "$tmp/site/templates/bad.html"
cat > "$tmp/site/templates/bad.html" <<'HTML'
<script src="https://example.com/tracker.js"></script>
HTML
assert_fail "external script blocked"

# Test 6: removed bad file, allowlisted scripts remain → pass
rm "$tmp/site/templates/bad.html"
assert_pass "after cleanup"

echo
echo "Results: $pass passed, $fail failed"
[[ "$fail" -eq 0 ]]
```

- [ ] **Step 2: Make it executable**

```bash
chmod +x /Users/joe/git/marina-alta-college/scripts/check-no-js_test.sh
```

- [ ] **Step 3: Write the allowlist**

Path: `/Users/joe/git/marina-alta-college/scripts/no-js-allowlist.txt`

```
# Lines starting with # are comments. Each non-comment line is a regex
# matched against the FULL <script ...> opening tag. If any allowlist
# regex matches, the tag is allowed.

# HTMX CDN (must remain at the canonical unpkg URL).
^<script src="https://unpkg\.com/htmx\.org

# Structured data (JSON-LD) — type="application/ld+json" is data, not behaviour.
type="application/ld\+json"

# Future: add a Plausible / analytics line here when adopted.
```

- [ ] **Step 4: Run the test — expect FAIL**

```bash
bash /Users/joe/git/marina-alta-college/scripts/check-no-js_test.sh
```
Expected: errors because `check-no-js.sh` doesn't exist yet.

- [ ] **Step 5: Write the script**

Path: `/Users/joe/git/marina-alta-college/scripts/check-no-js.sh`

```bash
#!/usr/bin/env bash
# Fails if any <script> tag in the given site dir is not on the allowlist.
# Args: <site-dir> [<allowlist-path>]
# Default allowlist: scripts/no-js-allowlist.txt (relative to the script's location).
set -euo pipefail

site_dir="${1:-site}"
allowlist="${2:-$(dirname "$0")/no-js-allowlist.txt}"

if [[ ! -d "$site_dir" ]]; then
  echo "check-no-js: site dir not found: $site_dir" >&2
  exit 2
fi

if [[ ! -f "$allowlist" ]]; then
  echo "check-no-js: allowlist not found: $allowlist" >&2
  exit 2
fi

# Build the regex from non-comment, non-blank lines of the allowlist.
allow_patterns=()
while IFS= read -r line; do
  [[ -z "$line" ]] && continue
  [[ "$line" =~ ^# ]] && continue
  allow_patterns+=("$line")
done < "$allowlist"

# Find every <script ...> opening tag in templates and static.
violations=0
while IFS= read -r match; do
  file="${match%%:*}"
  rest="${match#*:}"
  lineno="${rest%%:*}"
  content="${rest#*:}"

  # If any allow pattern matches, skip.
  allowed=0
  for pat in "${allow_patterns[@]}"; do
    if echo "$content" | grep -qE "$pat"; then
      allowed=1
      break
    fi
  done

  if [[ "$allowed" -eq 0 ]]; then
    echo "check-no-js: BLOCK $file:$lineno: $content" >&2
    violations=$((violations + 1))
  fi
done < <(grep -rEn '<script[[:space:]>]' "$site_dir/templates" "$site_dir/static" 2>/dev/null || true)

if [[ "$violations" -gt 0 ]]; then
  echo "check-no-js: $violations violation(s) found." >&2
  exit 1
fi

echo "check-no-js: OK"
```

- [ ] **Step 6: Make it executable**

```bash
chmod +x /Users/joe/git/marina-alta-college/scripts/check-no-js.sh
```

- [ ] **Step 7: Run the test — expect PASS**

```bash
bash /Users/joe/git/marina-alta-college/scripts/check-no-js_test.sh
```
Expected: `Results: 6 passed, 0 failed`.

- [ ] **Step 8: Run against the real `site/`**

```bash
cd /Users/joe/git/marina-alta-college
scripts/check-no-js.sh site
```
Expected: `OK`. If it reports a violation, investigate — it's either a real violation we need to fix or a missing allowlist entry.

- [ ] **Step 9: Commit**

```bash
git add scripts/check-no-js.sh scripts/no-js-allowlist.txt scripts/check-no-js_test.sh
git commit -m "feat: add no-js validation script + tests"
```

### Task F2: Write `scripts/check-i18n.go` (TDD)

**Files:**
- Create: `/Users/joe/git/marina-alta-college/scripts/go.mod`
- Create: `/Users/joe/git/marina-alta-college/scripts/check-i18n.go`
- Create: `/Users/joe/git/marina-alta-college/scripts/check-i18n_test.go`

- [ ] **Step 1: Initialize a Go module for scripts**

```bash
cd /Users/joe/git/marina-alta-college/scripts
go mod init github.com/marina-alta-college/marina-alta-college/scripts
```
Expected: `scripts/go.mod` created.

- [ ] **Step 2: Write the failing test**

Path: `/Users/joe/git/marina-alta-college/scripts/check-i18n_test.go`

```go
package main

import (
	"os"
	"path/filepath"
	"testing"
)

func writeJSON(t *testing.T, dir, name, content string) {
	t.Helper()
	p := filepath.Join(dir, name)
	if err := os.WriteFile(p, []byte(content), 0644); err != nil {
		t.Fatalf("write %s: %v", p, err)
	}
}

func TestCheckLocales_AllMatch(t *testing.T) {
	dir := t.TempDir()
	for _, lang := range []string{"en", "es", "de", "nl"} {
		writeJSON(t, dir, lang+".json", `{"hello":"x","goodbye":"y"}`)
	}
	missing, err := checkLocales(dir, []string{"en", "es", "de", "nl"})
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if len(missing) != 0 {
		t.Fatalf("expected no missing keys, got %v", missing)
	}
}

func TestCheckLocales_MissingKeyInOne(t *testing.T) {
	dir := t.TempDir()
	writeJSON(t, dir, "en.json", `{"hello":"x","goodbye":"y"}`)
	writeJSON(t, dir, "es.json", `{"hello":"x"}`)
	writeJSON(t, dir, "de.json", `{"hello":"x","goodbye":"y"}`)
	writeJSON(t, dir, "nl.json", `{"hello":"x","goodbye":"y"}`)

	missing, err := checkLocales(dir, []string{"en", "es", "de", "nl"})
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if len(missing) != 1 || missing[0].Locale != "es" || missing[0].Key != "goodbye" {
		t.Fatalf("expected missing es:goodbye, got %v", missing)
	}
}

func TestCheckLocales_NestedKeys(t *testing.T) {
	dir := t.TempDir()
	writeJSON(t, dir, "en.json", `{"nav":{"home":"Home","about":"About"}}`)
	writeJSON(t, dir, "es.json", `{"nav":{"home":"Inicio"}}`)
	writeJSON(t, dir, "de.json", `{"nav":{"home":"H","about":"A"}}`)
	writeJSON(t, dir, "nl.json", `{"nav":{"home":"H","about":"A"}}`)

	missing, err := checkLocales(dir, []string{"en", "es", "de", "nl"})
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if len(missing) != 1 || missing[0].Locale != "es" || missing[0].Key != "nav.about" {
		t.Fatalf("expected missing es:nav.about, got %v", missing)
	}
}

func TestCheckLocales_FileMissing(t *testing.T) {
	dir := t.TempDir()
	writeJSON(t, dir, "en.json", `{}`)
	// no es/de/nl
	_, err := checkLocales(dir, []string{"en", "es", "de", "nl"})
	if err == nil {
		t.Fatalf("expected error for missing locale file, got nil")
	}
}
```

- [ ] **Step 3: Run the test — expect FAIL**

```bash
cd /Users/joe/git/marina-alta-college/scripts
go test ./...
```
Expected: compile errors because `checkLocales` and `Missing` don't exist.

- [ ] **Step 4: Write the implementation**

Path: `/Users/joe/git/marina-alta-college/scripts/check-i18n.go`

```go
// check-i18n verifies that all locale JSON files share the same key set.
// Usage: go run check-i18n.go [<locales-dir>] (default: ../site/locales)
package main

import (
	"encoding/json"
	"fmt"
	"os"
	"path/filepath"
	"sort"
	"strings"
)

type Missing struct {
	Locale string
	Key    string
}

// flatten walks a JSON object and returns dotted keys for every leaf.
func flatten(prefix string, v interface{}, out map[string]struct{}) {
	switch t := v.(type) {
	case map[string]interface{}:
		for k, child := range t {
			next := k
			if prefix != "" {
				next = prefix + "." + k
			}
			flatten(next, child, out)
		}
	default:
		out[prefix] = struct{}{}
	}
}

func loadKeys(path string) (map[string]struct{}, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		return nil, err
	}
	var v interface{}
	if err := json.Unmarshal(data, &v); err != nil {
		return nil, fmt.Errorf("%s: %w", path, err)
	}
	out := map[string]struct{}{}
	flatten("", v, out)
	return out, nil
}

// checkLocales returns the list of (locale, key) pairs missing from any of
// the given locale JSONs in dir, using the union of keys across all locales
// as the target set.
func checkLocales(dir string, locales []string) ([]Missing, error) {
	keysByLocale := map[string]map[string]struct{}{}
	for _, lang := range locales {
		path := filepath.Join(dir, lang+".json")
		ks, err := loadKeys(path)
		if err != nil {
			return nil, err
		}
		keysByLocale[lang] = ks
	}

	// Union of all keys.
	union := map[string]struct{}{}
	for _, ks := range keysByLocale {
		for k := range ks {
			union[k] = struct{}{}
		}
	}

	var missing []Missing
	allKeys := make([]string, 0, len(union))
	for k := range union {
		allKeys = append(allKeys, k)
	}
	sort.Strings(allKeys)

	sortedLocales := append([]string(nil), locales...)
	sort.Strings(sortedLocales)

	for _, k := range allKeys {
		for _, lang := range sortedLocales {
			if _, ok := keysByLocale[lang][k]; !ok {
				missing = append(missing, Missing{Locale: lang, Key: k})
			}
		}
	}
	return missing, nil
}

func main() {
	dir := "../site/locales"
	if len(os.Args) >= 2 {
		dir = os.Args[1]
	}
	locales := []string{"en", "es", "de", "nl"}
	missing, err := checkLocales(dir, locales)
	if err != nil {
		fmt.Fprintf(os.Stderr, "check-i18n: %v\n", err)
		os.Exit(2)
	}
	if len(missing) > 0 {
		// Group by locale for nicer output.
		byLocale := map[string][]string{}
		for _, m := range missing {
			byLocale[m.Locale] = append(byLocale[m.Locale], m.Key)
		}
		fmt.Fprintln(os.Stderr, "check-i18n: missing keys:")
		for lang, keys := range byLocale {
			sort.Strings(keys)
			fmt.Fprintf(os.Stderr, "  %s: %s\n", lang, strings.Join(keys, ", "))
		}
		os.Exit(1)
	}
	fmt.Println("check-i18n: OK")
}
```

- [ ] **Step 5: Run the test — expect PASS**

```bash
cd /Users/joe/git/marina-alta-college/scripts
go test ./...
```
Expected: `PASS` for all four tests.

- [ ] **Step 6: Run against the real `site/locales/`**

```bash
cd /Users/joe/git/marina-alta-college/scripts
go run check-i18n.go ../site/locales
```
Expected: `OK`, or output of any real missing-key issues. If issues are flagged, fix the locale files before continuing.

- [ ] **Step 7: Commit**

```bash
cd /Users/joe/git/marina-alta-college
git add scripts/go.mod scripts/check-i18n.go scripts/check-i18n_test.go
git commit -m "feat: add i18n completeness validation script + tests"
```

### Task F3: Write `scripts/check-schema-sync.go` (TDD)

**Files:**
- Create: `/Users/joe/git/marina-alta-college/scripts/check-schema-sync.go`
- Create: `/Users/joe/git/marina-alta-college/scripts/check-schema-sync_test.go`

This script is more complex — it needs to parse Sanity schema TypeScript and Go templates. We use a pragmatic approach: regex-extract field names rather than build full parsers. The point is *consistency catches*, not perfect AST analysis.

- [ ] **Step 1: Write the failing test**

Path: `/Users/joe/git/marina-alta-college/scripts/check-schema-sync_test.go`

```go
package main

import (
	"os"
	"path/filepath"
	"testing"
)

func writeFile(t *testing.T, path, content string) {
	t.Helper()
	if err := os.MkdirAll(filepath.Dir(path), 0755); err != nil {
		t.Fatalf("mkdir: %v", err)
	}
	if err := os.WriteFile(path, []byte(content), 0644); err != nil {
		t.Fatalf("write %s: %v", path, err)
	}
}

func TestExtractSchemaFields_Simple(t *testing.T) {
	dir := t.TempDir()
	writeFile(t, filepath.Join(dir, "homePage.ts"), `
import {defineType, defineField} from 'sanity'

export const homePage = defineType({
  name: 'homePage',
  type: 'document',
  fields: [
    defineField({name: 'heroTitle', type: 'string'}),
    defineField({name: 'heroSubtitle', type: 'string'}),
    defineField({name: 'fastFacts', type: 'array', of: [{type: 'block'}]}),
  ],
})
`)
	fields, err := extractSchemaFields(dir)
	if err != nil {
		t.Fatalf("err: %v", err)
	}
	for _, want := range []string{"heroTitle", "heroSubtitle", "fastFacts"} {
		if _, ok := fields[want]; !ok {
			t.Errorf("expected field %q in extracted set, got %v", want, fields)
		}
	}
}

func TestExtractTemplateFields_Simple(t *testing.T) {
	dir := t.TempDir()
	writeFile(t, filepath.Join(dir, "home.html"), `
<h1>{{ .Home.HeroTitle }}</h1>
<p>{{ .Home.HeroSubtitle }}</p>
{{ range .Home.FastFacts }}
  <li>{{ .Label }}: {{ .Value }}</li>
{{ end }}
`)
	fields, err := extractTemplateFields(dir)
	if err != nil {
		t.Fatalf("err: %v", err)
	}
	for _, want := range []string{"HeroTitle", "HeroSubtitle", "FastFacts", "Label", "Value"} {
		if _, ok := fields[want]; !ok {
			t.Errorf("expected field %q in extracted set, got %v", want, fields)
		}
	}
}

func TestSchemaTemplateMismatch_TemplateReferencesUnknown(t *testing.T) {
	schemaDir := t.TempDir()
	tmplDir := t.TempDir()
	writeFile(t, filepath.Join(schemaDir, "homePage.ts"),
		`defineField({name: 'heroTitle', type: 'string'})`)
	writeFile(t, filepath.Join(tmplDir, "home.html"),
		`{{ .Home.HeroTitle }} {{ .Home.NonexistentField }}`)

	mismatches, err := compareFields(schemaDir, tmplDir)
	if err != nil {
		t.Fatalf("err: %v", err)
	}
	if len(mismatches) != 1 || mismatches[0].Field != "NonexistentField" {
		t.Fatalf("expected 1 mismatch on NonexistentField, got %v", mismatches)
	}
}
```

- [ ] **Step 2: Run the test — expect FAIL**

```bash
cd /Users/joe/git/marina-alta-college/scripts
go test -run TestExtractSchema ./...
```
Expected: undefined `extractSchemaFields` etc.

- [ ] **Step 3: Write the implementation**

Path: `/Users/joe/git/marina-alta-college/scripts/check-schema-sync.go`

```go
// check-schema-sync verifies that every Sanity-derived field referenced in
// Go templates exists in studio/schemaTypes/, and (best-effort) flags
// schema fields that are never rendered.
//
// Heuristics:
//   - Schema fields: regex matches `name: '<ident>'` inside studio/schemaTypes/*.ts.
//   - Template fields: regex matches `.HomePageStruct.<Ident>` patterns inside
//     site/templates/**/*.html. We strip the leading dot and the leading struct
//     prefix (e.g. .Home, .Settings, .Page) and treat the trailing identifier
//     as the field name to check.
//
// This is intentionally pragmatic, not a full TS or Go template parser. False
// positives are tolerable; false negatives (missed mismatches that hit prod)
// are not. Tune the heuristics over time as new patterns appear.
//
// Usage: go run check-schema-sync.go [<schema-dir>] [<templates-dir>]
//        (defaults: ../studio/schemaTypes, ../site/templates)
package main

import (
	"fmt"
	"io/fs"
	"os"
	"path/filepath"
	"regexp"
	"sort"
	"strings"
)

type Mismatch struct {
	File  string
	Line  int
	Field string
}

var (
	schemaFieldRe   = regexp.MustCompile(`name:\s*['"]([A-Za-z_][A-Za-z0-9_]*)['"]`)
	// .Word.Word — captures the trailing identifier as the field.
	templateFieldRe = regexp.MustCompile(`\.[A-Z][A-Za-z0-9_]*\.([A-Z][A-Za-z0-9_]*)`)
)

func extractSchemaFields(dir string) (map[string]struct{}, error) {
	out := map[string]struct{}{}
	err := filepath.WalkDir(dir, func(path string, d fs.DirEntry, err error) error {
		if err != nil {
			return err
		}
		if d.IsDir() || !strings.HasSuffix(path, ".ts") {
			return nil
		}
		data, err := os.ReadFile(path)
		if err != nil {
			return err
		}
		for _, m := range schemaFieldRe.FindAllStringSubmatch(string(data), -1) {
			out[m[1]] = struct{}{}
		}
		return nil
	})
	return out, err
}

func extractTemplateFields(dir string) (map[string]struct{}, error) {
	out := map[string]struct{}{}
	err := filepath.WalkDir(dir, func(path string, d fs.DirEntry, err error) error {
		if err != nil {
			return err
		}
		if d.IsDir() || !strings.HasSuffix(path, ".html") {
			return nil
		}
		data, err := os.ReadFile(path)
		if err != nil {
			return err
		}
		for _, m := range templateFieldRe.FindAllStringSubmatch(string(data), -1) {
			out[m[1]] = struct{}{}
		}
		return nil
	})
	return out, err
}

// canonicalize lowercases the first letter (Go template identifiers are
// PascalCase whereas Sanity schema field names are camelCase).
func canonicalize(s string) string {
	if s == "" {
		return s
	}
	return strings.ToLower(s[:1]) + s[1:]
}

func compareFields(schemaDir, tmplDir string) ([]Mismatch, error) {
	schema, err := extractSchemaFields(schemaDir)
	if err != nil {
		return nil, err
	}
	tmpl, err := extractTemplateFields(tmplDir)
	if err != nil {
		return nil, err
	}

	// Allowlist of struct/helper fields that aren't Sanity-backed.
	skip := map[string]struct{}{
		// Common PageData helpers + Go template iteration variables.
		"Lang": {}, "T": {}, "Settings": {}, "Page": {}, "Home": {},
		"Title": {}, "Slug": {}, "URL": {}, "Path": {}, "Index": {},
		"Items": {}, "Label": {}, "Value": {}, "Children": {}, "Lines": {},
	}

	var out []Mismatch
	keys := make([]string, 0, len(tmpl))
	for k := range tmpl {
		keys = append(keys, k)
	}
	sort.Strings(keys)

	for _, k := range keys {
		if _, ok := skip[k]; ok {
			continue
		}
		canon := canonicalize(k)
		if _, ok := schema[canon]; !ok {
			out = append(out, Mismatch{Field: k})
		}
	}
	return out, nil
}

func main() {
	schemaDir := "../studio/schemaTypes"
	tmplDir := "../site/templates"
	if len(os.Args) >= 2 {
		schemaDir = os.Args[1]
	}
	if len(os.Args) >= 3 {
		tmplDir = os.Args[2]
	}
	miss, err := compareFields(schemaDir, tmplDir)
	if err != nil {
		fmt.Fprintf(os.Stderr, "check-schema-sync: %v\n", err)
		os.Exit(2)
	}
	if len(miss) > 0 {
		fmt.Fprintln(os.Stderr, "check-schema-sync: template fields without matching schema field:")
		for _, m := range miss {
			fmt.Fprintf(os.Stderr, "  %s\n", m.Field)
		}
		fmt.Fprintln(os.Stderr, "If a flagged field is intentionally not Sanity-backed (Go-side helper), add it to the skip list in scripts/check-schema-sync.go.")
		os.Exit(1)
	}
	fmt.Println("check-schema-sync: OK")
}
```

- [ ] **Step 4: Run the test — expect PASS**

```bash
cd /Users/joe/git/marina-alta-college/scripts
go test ./...
```
Expected: all tests pass (i18n + schema-sync).

- [ ] **Step 5: Run against the real schema/templates**

```bash
cd /Users/joe/git/marina-alta-college/scripts
go run check-schema-sync.go ../studio/schemaTypes ../site/templates
```
Expected: `OK`, or a list of fields the heuristic doesn't recognize. For the first run, expect 0-3 false positives — add them to the `skip` allowlist in the source. The point is to make the gate green on the *current* codebase before turning on enforcement.

- [ ] **Step 6: If false positives appeared, expand the skip list**

Edit `scripts/check-schema-sync.go`'s `skip` map to include real Go-side helpers. Re-run the test (`go test ./...`) and the real-codebase check (`go run check-schema-sync.go ...`) until both pass.

- [ ] **Step 7: Commit**

```bash
cd /Users/joe/git/marina-alta-college
git add scripts/check-schema-sync.go scripts/check-schema-sync_test.go
git commit -m "feat: add schema-sync validation script + tests"
```

---

## Phase G — Makefile + CI Workflows

### Task G1: Write the top-level `Makefile`

**Files:**
- Create: `/Users/joe/git/marina-alta-college/Makefile`

- [ ] **Step 1: Write the Makefile**

Path: `/Users/joe/git/marina-alta-college/Makefile`

```makefile
.PHONY: check check-build check-no-js check-schema-sync check-i18n check-go-test help

help:
	@echo "make check    - run the full local validation gate"
	@echo "make help     - this message"

check: check-build check-no-js check-schema-sync check-i18n check-go-test
	@echo "make check: ALL GREEN"

check-build:
	@echo "→ go build site/..."
	@cd site && go build ./...
	@echo "→ go vet site/..."
	@cd site && go vet ./...
	@echo "→ studio build"
	@cd studio && npm run build

check-no-js:
	@echo "→ no-js guard"
	@scripts/check-no-js.sh site

check-schema-sync:
	@echo "→ schema-sync"
	@cd scripts && go run check-schema-sync.go ../studio/schemaTypes ../site/templates

check-i18n:
	@echo "→ i18n completeness"
	@cd scripts && go run check-i18n.go ../site/locales

check-go-test:
	@echo "→ go test scripts/..."
	@cd scripts && go test ./...
```

- [ ] **Step 2: Run `make check` to verify the local gate is green on the current tree**

```bash
cd /Users/joe/git/marina-alta-college
make check
```
Expected: `make check: ALL GREEN`. Any failure means a script needs adjustment OR the codebase has a real problem to fix before turning on the gate.

- [ ] **Step 3: Commit**

```bash
git add Makefile
git commit -m "feat: add top-level Makefile (make check runs the local gate)"
```

### Task G2: Write `.github/workflows/ci.yml`

**Files:**
- Create: `/Users/joe/git/marina-alta-college/.github/workflows/ci.yml`

- [ ] **Step 1: Write the workflow**

Path: `/Users/joe/git/marina-alta-college/.github/workflows/ci.yml`

```yaml
name: ci
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  contents: read
  pull-requests: write

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      site: ${{ steps.filter.outputs.site }}
      studio: ${{ steps.filter.outputs.studio }}
      scripts: ${{ steps.filter.outputs.scripts }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            site:
              - 'site/**'
              - 'scripts/**'
              - 'Makefile'
              - '.github/workflows/ci.yml'
            studio:
              - 'studio/**'
              - 'scripts/**'
              - 'Makefile'
              - '.github/workflows/ci.yml'
            scripts:
              - 'scripts/**'

  check:
    needs: detect-changes
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - uses: actions/setup-node@v4
        if: needs.detect-changes.outputs.studio == 'true'
        with:
          node-version: '20'
      - name: studio install
        if: needs.detect-changes.outputs.studio == 'true'
        run: cd studio && npm ci
      - name: studio build
        if: needs.detect-changes.outputs.studio == 'true'
        run: cd studio && npm run build
      - name: site build
        if: needs.detect-changes.outputs.site == 'true'
        run: cd site && go build ./...
      - name: site vet
        if: needs.detect-changes.outputs.site == 'true'
        run: cd site && go vet ./...
      - name: no-js guard
        run: scripts/check-no-js.sh site
      - name: schema-sync
        run: cd scripts && go run check-schema-sync.go ../studio/schemaTypes ../site/templates
      - name: i18n completeness
        run: cd scripts && go run check-i18n.go ../site/locales
      - name: scripts tests
        if: needs.detect-changes.outputs.scripts == 'true'
        run: cd scripts && go test ./...
```

- [ ] **Step 2: Push to a feature branch and verify the workflow runs**

```bash
cd /Users/joe/git/marina-alta-college
git checkout -b ci-bootstrap
git add .github/workflows/ci.yml
git commit -m "feat: add ci workflow"
git push -u origin ci-bootstrap
gh pr create --title "ci: bootstrap workflow" --body "Bootstrap CI. See plan." --base main --head ci-bootstrap
```
Expected: a PR opens. After ~2 min, `gh pr checks <pr> --repo marina-alta-college/marina-alta-college` shows `ci / check  pass`.

- [ ] **Step 3: Merge the PR (this one needs human merge — branch protection isn't on yet)**

```bash
gh pr merge --squash --delete-branch
```
Expected: PR merged, branch deleted, you're back on main.

### Task G3: Write `.github/workflows/post-deploy.yml`

**Files:**
- Create: `/Users/joe/git/marina-alta-college/.github/workflows/post-deploy.yml`

- [ ] **Step 1: Write the workflow**

Path: `/Users/joe/git/marina-alta-college/.github/workflows/post-deploy.yml`

```yaml
name: post-deploy
on:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  detect-studio-changes:
    runs-on: ubuntu-latest
    outputs:
      studio: ${{ steps.filter.outputs.studio }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            studio:
              - 'studio/**'

  studio-deploy:
    needs: detect-studio-changes
    if: needs.detect-studio-changes.outputs.studio == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - name: install
        run: cd studio && npm ci
      - name: deploy schema
        run: cd studio && npx sanity schema deploy
        env:
          SANITY_AUTH_TOKEN: ${{ secrets.SANITY_AUTH_TOKEN }}
      - name: deploy studio
        run: cd studio && npx sanity deploy
        env:
          SANITY_AUTH_TOKEN: ${{ secrets.SANITY_AUTH_TOKEN }}

  smoke-check:
    needs: detect-studio-changes
    runs-on: ubuntu-latest
    steps:
      # Wait for Render to finish rebuilding (typical: 2-3 min after push).
      - name: wait for render
        run: sleep 240
      - name: smoke check
        run: |
          set -e
          status=$(curl -sS -o /dev/null -w '%{http_code}' https://marinaaltacollege.com)
          if [ "$status" != "200" ]; then
            echo "Smoke check failed: HTTP $status"
            exit 1
          fi
          echo "Smoke check OK"
```

- [ ] **Step 2: Add the Sanity auth token as a GitHub Secret**

Action for the user (the agent should pause and ask): in the GitHub repo settings → Secrets and variables → Actions, add a new secret named `SANITY_AUTH_TOKEN`. Get the token value from `https://www.sanity.io/manage/personal/tokens` (create a new "Deploy Studio" token with deploy-studio + deploy-graphql scopes).

After confirming the secret exists:

```bash
gh secret list --repo marina-alta-college/marina-alta-college
```
Expected: includes `SANITY_AUTH_TOKEN`.

- [ ] **Step 3: Commit and push (no PR needed — workflow files only run on subsequent pushes)**

```bash
cd /Users/joe/git/marina-alta-college
git add .github/workflows/post-deploy.yml
git commit -m "feat: add post-deploy workflow (studio deploy + smoke check)"
git push origin main
```
Expected: push succeeds (we don't have branch protection yet). The next time `studio/**` changes on main, the studio-deploy job runs.

---

## Phase H — Branch Protection

### Task H1: Configure branch protection on main

**Files:** none (GitHub-side configuration via API).

- [ ] **Step 1: Set the protection rules**

```bash
gh api -X PUT repos/marina-alta-college/marina-alta-college/branches/main/protection \
  -F required_status_checks.strict=true \
  -F 'required_status_checks.contexts[]=ci / check' \
  -F enforce_admins=false \
  -F required_pull_request_reviews=null \
  -F restrictions=null \
  -F allow_force_pushes=false \
  -F allow_deletions=false
```
Expected: API returns the protection config JSON.

- [ ] **Step 2: Enable auto-merge on the repo**

```bash
gh api -X PATCH repos/marina-alta-college/marina-alta-college \
  -F allow_auto_merge=true \
  -F allow_squash_merge=true \
  -F delete_branch_on_merge=true
```
Expected: returns the repo config with `allow_auto_merge: true`.

- [ ] **Step 3: Verify**

```bash
gh api repos/marina-alta-college/marina-alta-college --jq '.allow_auto_merge, .allow_squash_merge, .delete_branch_on_merge'
gh api repos/marina-alta-college/marina-alta-college/branches/main/protection --jq '.required_status_checks'
```
Expected: three `true`s, then the required-checks object showing `ci / check`.

- [ ] **Step 4: Sanity-check by trying to push to main directly**

```bash
cd /Users/joe/git/marina-alta-college
git commit --allow-empty -m "test: should-be-rejected"
git push origin main
```
Expected: rejection (branch protection requires PR + check). Then revert the local commit:

```bash
git reset --hard HEAD~1
```

---

## Phase I — End-to-End Dry Run

### Task I1: Author and ship a smoke-test spec

The point of this phase is to prove the whole pipeline works on a tiny, low-risk change before throwing real walkthroughs at it.

**Files:**
- Create: `/Users/joe/git/marina-alta-college/docs/specs/2026-05-05-tagline-field.md`

- [ ] **Step 1: Write the smoke-test spec**

Path: `/Users/joe/git/marina-alta-college/docs/specs/2026-05-05-tagline-field.md`

```markdown
# tagline-field: add a tagline shown beneath the home hero

## Overview
Add an optional `tagline` field to the `siteSettings` Sanity singleton, displayed as italic supplementary text beneath the home page hero subtitle. Used to surface short editorial taglines (e.g. "A college for people who want to think for themselves").

## Source
- Authored as a smoke-test for the agentic pipeline. Not from a client transcript.

## Acceptance criteria
- `siteSettings` schema has a new optional `tagline` field with per-language variants.
- Home page renders the tagline below the existing hero subtitle when set.
- Hero renders correctly when the field is empty (no extra whitespace, no broken markup).
- Stub data in `handlers.go` provides a sensible default tagline for offline dev.
- All four locale files have the same key set (no new UI strings expected, since the tagline content lives in Sanity, not in locales).

## Routes affected
- `/{lang}` (home page)

## Schema changes
- `studio/schemaTypes/siteSettings.ts`: add `tagline` field, type `object` with `en`, `es`, `de`, `nl` string children, all optional.

## Content updates
- Seed `siteSettings` document with a placeholder tagline in `en`. Other locales empty (will fall back to en).

## Visual notes
- Italic, slightly smaller than subtitle, indented under it. Don't over-style — this is a quiet editorial flourish, not a callout.

## Out of scope
- Marketing copy choices for the actual tagline text.
- Per-page tagline overrides.
```

- [ ] **Step 2: File the issue + apply triage labels manually**

```bash
gh issue create \
  --repo marina-alta-college/marina-alta-college \
  --title "tagline-field: add tagline shown beneath the home hero" \
  --body "Spec: \`docs/specs/2026-05-05-tagline-field.md\`. Smoke-test for the /implement pipeline." \
  --label "enhancement,ready-for-agent,kind:tweak"
```
Note: this is `kind:tweak` (not `kind:new-section`), so `design-explorer` will be skipped.

- [ ] **Step 3: Commit the spec**

```bash
cd /Users/joe/git/marina-alta-college
git add docs/specs/2026-05-05-tagline-field.md
git commit -m "docs: add smoke-test spec (tagline-field)"
git push origin main
```

- [ ] **Step 4: Run `/implement tagline-field` in this session**

The orchestrating session reads `.claude/commands/implement.md` and executes the pipeline.

Expected sequence:
1. Pre-flight passes (spec exists, kind detected, no `needs-info`).
2. Worktree opens at `/Users/joe/git/marina-alta-trees/tagline-field`, branch `feat/tagline-field`.
3. `builder` runs: schema added, GROQ query updated, model updated, handler updated, template updated, locales unchanged (no new UI strings), stub updated, CSS unchanged.
4. `design-critic` runs Playwright against `http://localhost:8080/en` at desktop + mobile, captures screenshots, returns PASS.
5. `qa-reviewer` runs `make check` (ALL GREEN), opens PR with `automerge` label, enables auto-merge.
6. CI runs in ~2 min, merges automatically.
7. Render rebuilds the site.
8. `post-deploy.yml` runs studio deploy + smoke check.

- [ ] **Step 5: Verify the merged result**

After the merge:

```bash
cd /Users/joe/git/marina-alta-college
git checkout main && git pull
gh pr list --repo marina-alta-college/marina-alta-college --state merged --limit 1
curl -s https://marinaaltacollege.com | grep -i "tagline" || echo "(no live render yet — Sanity content still empty)"
```

The HTML may not yet show the tagline because the Sanity singleton hasn't been seeded with content. Seed it:

```bash
cd studio && npx sanity documents replace --replace-existing siteSettings - <<'EOF'
{ "_id": "siteSettings", "_type": "siteSettings", "tagline": { "en": "A college for people who want to think for themselves." } }
EOF
```

Then `curl -s https://marinaaltacollege.com | grep -i "think for themselves"` should match.

- [ ] **Step 6: Append the dry-run outcome to the agent log**

```bash
cd /Users/joe/git/marina-alta-college
git pull origin main  # in case post-deploy added commits
# Add a docs/agent-log.md block summarizing what worked / didn't.
# Then commit and push.
```

If anything in the chain failed: investigate, fix, re-run. The dry-run is the acceptance test for the entire installation.

---

## Self-review

I (the plan-writer) ran the following checks on this plan:

**1. Spec coverage:**
- Phase A covers the spec's "Phase A — Make the parent dir a repo + consolidate" ✓
- Phase B covers the spec's mattpocock skills install + docs/agents/ scaffolding ✓
- Phases C-E cover the spec's `.claude/` scaffolding (agents, commands, team templates, hooks, settings) ✓
- Phase F covers the validation scripts ✓
- Phase G covers the Makefile + CI workflows ✓
- Phase H covers branch protection + auto-merge enablement ✓
- Phase I covers the end-to-end dry run ✓
- Sanity auth token secret addressed in G3 step 2 ✓
- Render dashboard switch addressed in A7 ✓

**2. Placeholder scan:** No "TBD", "TODO" (in plan instructions; the agent-log seed has none), "implement later", or "similar to Task N". Each task has full content.

**3. Type consistency:** Field/struct names referenced in agent definitions and the schema-sync test align. The `kind:` label set (`kind:new-section`, `kind:tweak`) is consistent across walkthrough.md, implement.md, triage-labels.md, and the spec.

**4. Known gaps / open items:**
- The exact Render dashboard step is human-driven (A7); plan documents this as an explicit pause point.
- `SANITY_AUTH_TOKEN` is human-driven (G3); plan documents this as a pause point too.
- The schema-sync skip list is bootstrapped from the *first* live run — the plan accommodates iteration on the allowlist (F3 step 6).

---

## Execution

Plan complete and saved to `docs/superpowers/plans/2026-05-05-agentic-workflow-setup.md`. Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration. Good for a long mechanical plan like this.

**2. Inline Execution** — Execute tasks in this session using `executing-plans`, batch execution with checkpoints for review.

Which approach?
