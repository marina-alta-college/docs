# SEO & AI Search Optimisation

The site is optimised for both traditional search engines and AI-powered search tools.

## Endpoints

All SEO endpoints are generated dynamically by `seo.go`:

| Endpoint | Purpose |
|---|---|
| `/robots.txt` | Search engine crawling directives |
| `/sitemap.xml` | Full sitemap with hreflang annotations |
| `/llms.txt` | Structured summary for AI search agents |
| `/llms-full.txt` | Complete site content in Markdown for AI ingestion |

## robots.txt

Allows all major search engines and AI crawlers (Google, Bing, ChatGPT, Claude, Perplexity, etc.). Blocks known scrapers (CCBot, Bytespider). Points to the sitemap.

## sitemap.xml

Auto-generated from the route list and cached blog posts. For each page:
- Generates entries for all 4 languages
- Includes `xhtml:link` hreflang cross-references
- Sets `x-default` to English

Blog post URLs are included dynamically from the cache.

## llms.txt

A machine-readable summary of the site following the [llms.txt specification](https://llmstxt.org/). Contains:
- Site description
- Page index with URLs
- Key facts and statistics
- Link to `llms-full.txt` for complete content

## llms-full.txt

Complete text content of all pages in Markdown format. AI agents can read this single file to understand everything about the college. Content is pulled from the live cache.

## HTML SEO

The `seo-head.html` partial (in `templates/partials/`) adds to every page:
- `<title>` and `<meta name="description">`
- Open Graph tags (`og:title`, `og:description`, `og:image`)
- `hreflang` link tags for all language variants
- Canonical URL
- JSON-LD structured data (where applicable)

## Updating SEO

When you add or remove a page:

1. Add the page to the `pages` slice in `handleSitemapXML()` in `seo.go`
2. Add a `[Page Name]` entry to `handleLlmsTxt()` in `seo.go`
3. Add content rendering to `handleLlmsFullTxt()` in `seo.go`
4. Add appropriate meta tags in the page template via the `seo-head.html` partial
