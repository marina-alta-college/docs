# Architecture Overview

## How the System Works

```
┌─────────────┐     GROQ over HTTPS      ┌──────────────────┐
│  Sanity CMS  │◄────────────────────────►│   Go Web Server  │
│  (Content)   │  (read at startup +      │   (site repo)    │
│              │   webhook refresh)       │                  │
└──────┬───────┘                          └────────┬─────────┘
       │                                           │
       │ sanity.studio                             │ Render
       │ (content editors)                         │ (marinaaltacollege.com)
       ▼                                           ▼
  ┌──────────┐                              ┌──────────┐
  │  Studio   │                              │ Browser  │
  │  (React)  │                              │ (HTML +  │
  │           │                              │  HTMX)   │
  └──────────┘                              └──────────┘
```

### Request Flow

1. User visits `marinaaltacollege.com/en/admissions`
2. Go server matches the route in `routes.go` → dispatches to `handleAdmissions` in `handlers.go`
3. Handler reads content from the **in-memory cache** (not a live API call)
4. Content + locale strings are combined into a `PageData` struct
5. Go's `html/template` renders the HTML using templates from `templates/`
6. Response is gzip-compressed and sent to the browser
7. HTMX handles any progressive enhancement (e.g. calendar month navigation)

### Content Flow

1. Content editor publishes a change in Sanity Studio
2. Sanity fires a webhook → `POST /webhook/sanity` on the Go server
3. The server validates the HMAC-SHA256 signature
4. `cache.Refresh()` fetches all content from Sanity (bypassing CDN for freshness)
5. New content is swapped atomically into the in-memory cache
6. Next request serves the updated content — no restart needed

### Startup Sequence

1. Load i18n JSON files from `locales/`
2. Parse all Go templates from `templates/`
3. Create Sanity HTTP client from env vars
4. Warm the content cache (fetch everything for all 4 languages)
5. Register routes and start HTTP server

## Key Design Decisions

### Why Go (not Next.js, Astro, etc.)?

- Zero runtime dependencies — single static binary
- Extremely fast cold starts on Render
- Go's `html/template` is secure by default (auto-escapes HTML)
- No JavaScript build pipeline complexity for the server

### Why HTMX (not React, Vue, etc.)?

- No JavaScript frameworks to maintain or upgrade
- Server renders all HTML — simpler mental model
- Progressive enhancement: pages work without JS
- Smaller page payloads, better performance

### Why Sanity (not WordPress, Contentful, etc.)?

- Structured content with a typed schema
- Real-time editing with live preview (Sanity Studio)
- GROQ query language is powerful and flexible
- Generous free tier for small projects
- Portable Text gives rich content without raw HTML in the CMS

### Why in-memory cache (not per-request API calls)?

- Pages load in <50ms regardless of Sanity API latency
- Webhook-driven refresh means content is typically fresh within seconds
- Fallback stub data means the site works even if Sanity is down
- No external cache infrastructure (Redis, etc.) needed

## File-by-File Guide (site repo)

| File | Responsibility |
|---|---|
| `main.go` | Entry point — loads translations, parses templates, creates Sanity client, starts server |
| `routes.go` | URL routing — maps paths to handlers, serves static files |
| `handlers.go` | HTTP handlers — one per page, plus language switching and stubs |
| `sanity.go` | Sanity API client — GROQ queries, Portable Text → HTML conversion |
| `cache.go` | In-memory content cache — per-language storage, webhook handler, HMAC validation |
| `models.go` | Data types — Go structs for every content type |
| `render.go` | Template rendering — `PageData` struct, template functions, `renderPage` helper |
| `i18n.go` | Internationalisation — loads JSON locales, `T()` lookup, language detection |
| `middleware.go` | HTTP middleware — gzip compression, cache-control headers |
| `seo.go` | SEO handlers — robots.txt, sitemap.xml, llms.txt, llms-full.txt |
| `Makefile` | Build commands — `make dev`, `make build`, `make css` |
| `render.yaml` | Render deployment config — build command, env vars, region |

## URL Structure

All pages follow the pattern `/{lang}/{page}`:

| URL | Page |
|---|---|
| `/` | Redirects to `/{detected-lang}` |
| `/{lang}` | Home |
| `/{lang}/what-we-offer` | What We Offer |
| `/{lang}/who-we-are` | Who We Are |
| `/{lang}/admissions` | Admissions & Fees |
| `/{lang}/contact` | Contact |
| `/{lang}/journal` | Journal listing |
| `/{lang}/journal/{slug}` | Journal post |
| `/{lang}/calendar` | Academic calendar |
| `/set-lang/{lang}` | Set language cookie + redirect |
| `/webhook/sanity` | Sanity webhook endpoint (POST) |
| `/robots.txt` | SEO |
| `/sitemap.xml` | SEO |
| `/llms.txt` | AI search summary |
| `/llms-full.txt` | AI search full content |

Supported languages: `en`, `es`, `de`, `nl`
