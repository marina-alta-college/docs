# Sanity Studio (CMS)

The CMS lives in the [`studio`](https://github.com/marina-alta-college/studio) repo and is deployed to https://marina-alta-college.sanity.studio.

## Prerequisites

- **Node.js** 18+
- **npm**

## Quick Start

```bash
git clone git@github.com:marina-alta-college/studio.git
cd studio
npm install
npm run dev
```

Studio runs at http://localhost:3333

## Commands

```bash
npm run dev           # Local development server
npm run build         # Build for production
npm run deploy        # Deploy to marina-alta-college.sanity.studio
npm run schema:deploy # Deploy schema to Sanity cloud
```

## Project Config

| Setting | Value |
|---|---|
| Project ID | `e8pjc321` |
| Dataset | `production` |
| Studio Host | `marina-alta-college` |
| Studio URL | https://marina-alta-college.sanity.studio |

Configuration is in `sanity.config.ts` and `sanity.cli.ts`.

## Content Structure

The studio has two kinds of documents:

### Singleton Pages (fixed document IDs)

These are single documents — there's exactly one of each:

| Document Type | Document ID | Purpose |
|---|---|---|
| `homePage` | `homePage` | Home page content |
| `whatWeOfferPage` | `whatWeOfferPage` | What We Offer page |
| `whoWeArePage` | `whoWeArePage` | Who We Are page |
| `admissionsPage` | `admissionsPage` | Admissions & Fees page |
| `contactPage` | `contactPage` | Contact page |
| `siteSettings` | `siteSettings` | Global settings (name, email, phone, address) |

These are configured as singletons in `structure.ts` — they appear as single items in the sidebar, not as lists.

### Collection Documents

These are lists of documents — you can create as many as needed:

| Document Type | Purpose |
|---|---|
| `blogPost` | Journal / blog posts |
| `calendarEvent` | Academic calendar events |

## Schema Files

All schemas are in `schemaTypes/`:

| File | Document Type |
|---|---|
| `homePage.ts` | Home page |
| `whatWeOfferPage.ts` | What We Offer |
| `whoWeArePage.ts` | Who We Are |
| `admissionsPage.ts` | Admissions & Fees |
| `contactPage.ts` | Contact |
| `siteSettings.ts` | Site Settings |
| `blogPost.ts` | Journal posts |
| `calendarEvent.ts` | Calendar events |
| `index.ts` | Exports all schemas |

## i18n Pattern

All translatable fields use a "field per language" pattern:

- `title` — English (base, always required)
- `title_es` — Spanish
- `title_de` — German
- `title_nl` — Dutch

This is implemented via helper functions in each schema file:

```typescript
// Creates: title (EN), title_es, title_de, title_nl
localizedString('title', 'Title', { required: true })

// Creates: body (EN), body_es, body_de, body_nl (Portable Text)
localizedBlockContent('body', 'Body')
```

English is always required. Other languages are optional and fall back to English on the website.

## Rich Content (Portable Text)

Body fields use Sanity's Portable Text — a structured rich text format. The blog post schema also supports inline images with alt text and captions.

The Go site converts Portable Text to HTML in `sanity.go` → `portableTextToHTML()`.

## Webhook

When content is published in the Studio, Sanity sends a webhook to the production site at:

```
POST https://marinaaltacollege.com/webhook/sanity
```

This triggers a cache refresh so content changes appear on the live site within seconds. The webhook is secured with HMAC-SHA256 signature validation.

To configure the webhook:
1. Go to https://www.sanity.io/manage/project/e8pjc321
2. Navigate to API → Webhooks
3. Set URL to `https://marinaaltacollege.com/webhook/sanity`
4. Set the secret (must match `SANITY_WEBHOOK_SECRET` env var on Render)
5. Trigger on: Create, Update, Delete
