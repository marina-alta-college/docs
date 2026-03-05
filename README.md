# Marina Alta College — Technical Documentation

Complete documentation for the Marina Alta College website infrastructure. This covers everything a developer needs to take over, maintain, and extend the system.

## Quick Links

| Document | What it covers |
|---|---|
| [Architecture Overview](./architecture.md) | System design, how the pieces fit together |
| [Site (Go Backend)](./site.md) | The production website — build, run, deploy, extend |
| [Sanity Studio (CMS)](./studio.md) | Content management — schemas, editing, deployment |
| [Content Editing Guide](./content-editing.md) | Non-technical guide for editing site content |
| [i18n (Translations)](./i18n.md) | How multi-language support works |
| [Deployment & Hosting](./deployment.md) | Render, DNS, environment variables, going live |
| [SEO & AI Search](./seo.md) | robots.txt, sitemap, llms.txt, structured data |
| [Design System](./design-system.md) | Maritime Clean theme, colours, typography, CSS |
| [How-To Guides](./how-to.md) | Step-by-step recipes for common tasks |
| [Legacy & Reference](./legacy.md) | Mood board, POC site, WordPress content exports |

## Repository Map

All repos live under the GitHub org **[marina-alta-college](https://github.com/marina-alta-college)**:

| Repo | Purpose | Tech | Deployed to |
|---|---|---|---|
| [`site`](https://github.com/marina-alta-college/site) | Production website | Go, HTMX, Tailwind 4, DaisyUI 5 | Render (Frankfurt) |
| [`studio`](https://github.com/marina-alta-college/studio) | Sanity CMS Studio | React, TypeScript, Sanity v3 | marina-alta-college.sanity.studio |
| [`docs`](https://github.com/marina-alta-college/docs) | This documentation | Markdown | — |
| [`mood-board`](https://github.com/marina-alta-college/mood-board) | Design direction picker (archived) | Go, HTMX | — |
| [`poc-site`](https://github.com/marina-alta-college/poc-site) | Early proof-of-concept (archived) | Go, HTMX | — |

## Key Credentials & Accounts

> **Important**: These are project identifiers, not secrets. Actual secrets (API tokens, webhook secrets) are stored as environment variables on Render and should never be committed to code.

| Service | Identifier |
|---|---|
| Sanity Project ID | `e8pjc321` |
| Sanity Dataset | `production` |
| Sanity Studio URL | https://marina-alta-college.sanity.studio |
| Render Service | `marina-alta-college` (Frankfurt region) |
| Domain | marinaaltacollege.com |

## Tech Stack Summary

- **Backend**: Go (standard library only — no frameworks)
- **Frontend**: Server-side rendered HTML via Go `html/template`
- **Interactivity**: HTMX 2 (no JavaScript frameworks, no custom JS)
- **Styling**: Tailwind CSS 4 + DaisyUI 5 (Maritime Clean theme)
- **CMS**: Sanity v3 (headless, content delivered via GROQ API)
- **Hosting**: Render (Go web service, starter plan)
- **Languages**: English, Spanish, German, Dutch
