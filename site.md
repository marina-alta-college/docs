# Site (Go Backend)

The production website lives in the [`site`](https://github.com/marina-alta-college/site) repo.

## Prerequisites

- **Go** 1.21+ (uses Go 1.22 routing features: `r.PathValue()`, method patterns)
- **Node.js** 18+ (for Tailwind CSS build)
- **npm** (comes with Node)

## Quick Start

```bash
# Clone
git clone git@github.com:marina-alta-college/site.git
cd site

# Install CSS build tools
npm install

# Create environment file
cp .env.example .env
# Edit .env if needed (defaults work for development)

# Run in development mode (CSS watcher + Go server)
make dev
```

The site will be available at http://localhost:8080

## Environment Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `PORT` | No | `8080` | HTTP server port |
| `SANITY_PROJECT_ID` | No | (empty) | Sanity project ID. If empty, stub data is used |
| `SANITY_DATASET` | No | `production` | Sanity dataset name |
| `SANITY_API_VERSION` | No | `2025-03-03` | Sanity API date version |
| `SANITY_USE_CDN` | No | `true` | Use Sanity CDN (faster, eventually consistent) |
| `SANITY_WEBHOOK_SECRET` | No | (empty) | HMAC secret for webhook signature validation |
| `BASE_URL` | No | `https://marinaaltacollege.com` | Canonical URL for sitemaps and SEO |

**For local development**, you only need `SANITY_PROJECT_ID=e8pjc321` in your `.env`. Everything else has sensible defaults. If you omit the project ID, the site runs with built-in stub content.

## Build Commands

```bash
make dev          # CSS watcher + Go server (development)
make css          # Build CSS once (minified)
make css-watch    # CSS watcher only
make build        # Build production binary (./site)
make clean        # Remove binary and CSS output
```

## Project Structure

```
site/
├── main.go              # Entry point
├── routes.go            # URL routing
├── handlers.go          # HTTP handlers (one per page + stubs)
├── sanity.go            # Sanity API client + Portable Text → HTML
├── cache.go             # In-memory content cache + webhook handler
├── models.go            # Go data types
├── render.go            # Template rendering + helper functions
├── i18n.go              # Translation loading + language detection
├── middleware.go         # Gzip + cache-control
├── seo.go               # robots.txt, sitemap, llms.txt
├── Makefile
├── render.yaml          # Render deployment config
├── .env.example
├── locales/
│   ├── en.json          # English UI strings
│   ├── es.json          # Spanish
│   ├── de.json          # German
│   └── nl.json          # Dutch
├── templates/
│   ├── base.html        # HTML shell (head, scripts, layout)
│   ├── partials/
│   │   ├── navbar.html
│   │   ├── footer.html
│   │   ├── logo.html
│   │   ├── contact-bubble.html  # WhatsApp floating button
│   │   └── seo-head.html        # Meta tags, Open Graph, JSON-LD
│   └── pages/
│       ├── home.html
│       ├── what-we-offer.html
│       ├── who-we-are.html
│       ├── admissions.html
│       ├── contact.html
│       ├── journal.html
│       ├── journal-post.html
│       ├── calendar.html
│       └── 404.html
├── static/
│   ├── css/
│   │   ├── input.css    # Tailwind source + Maritime Clean theme
│   │   └── output.css   # Built CSS (gitignored)
│   └── img/
│       ├── octopus.svg  # Logo SVG
│       └── favicon.svg
└── package.json         # Tailwind/DaisyUI npm deps
```

## How Pages Work

Every page follows the same pattern:

1. **Handler** (in `handlers.go`) reads content from the cache, falls back to stubs
2. **PageData** struct is populated with content, locale strings, and metadata
3. **renderPage** compiles the template and writes the response

```go
// Example: handlers.go
func handleAdmissions(w http.ResponseWriter, r *http.Request) {
    lang := r.PathValue("lang")
    admissions := cache.Admissions(lang)
    if admissions == nil {
        admissions = stubAdmissionsPage()
    }
    settings := cache.Settings(lang)
    if settings == nil {
        settings = stubSiteSettings()
    }
    renderPage(w, r, "admissions", &PageData{
        Title:      "Admissions — Marina Alta College",
        Lang:       lang,
        Path:       "/admissions",
        Page:       "admissions",
        Admissions: admissions,
        Settings:   settings,
    })
}
```

## Template System

Templates use Go's `html/template`. Key conventions:

- `base.html` defines the outer HTML shell with `{{block "content" .}}` for page content
- Each page template defines `{{define "content"}}...{{end}}`
- Partials are included via `{{template "navbar" .}}`
- Inside `{{range}}` loops, use `$` to access root PageData: `{{$.T}}`, `{{$.Settings}}`

### Template Functions

| Function | Usage | Description |
|---|---|---|
| `t` | `{{t .T "key"}}` | Look up a translation string |
| `safeHTML` | `{{safeHTML .Body}}` | Render trusted HTML from CMS |
| `formatDate` | `{{formatDate .PublishedAt .Lang}}` | Localised date |
| `langPath` | `{{langPath .Lang "/admissions"}}` | Build `/{lang}/path` URL |
| `isActivePath` | `{{isActivePath .Path "/admissions"}}` | Navbar active state |
| `even` | `{{if even $i}}` | Alternating row styles |
| `add` | `{{add $i 1}}` | Arithmetic in templates |

## Sanity Integration

The site talks to Sanity via its HTTP API (no SDK). See `sanity.go`.

- `SanityClient` is created from env vars at startup
- Each page has a `Fetch*` method that builds a GROQ query and parses the response
- Portable Text (rich content) is converted to HTML by `portableTextToHTML()`
- Images are served from `cdn.sanity.io`

### i18n in GROQ

Content fields have language variants (e.g. `title`, `title_es`, `title_de`, `title_nl`). The queries use `coalesce()` to fall back to English:

```groq
"title": coalesce(title_es, title)
```

This is handled by the `coalesceField()` helper in `sanity.go`.

## Stub Data

Every handler has a fallback stub function (e.g. `stubHomePage()`) in `handlers.go`. This means:

- The site runs without Sanity configured (useful for local dev)
- If Sanity goes down, the site still serves content
- Stubs contain real marketing copy — they're not lorem ipsum
