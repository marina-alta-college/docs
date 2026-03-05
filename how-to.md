# How-To Guides

Step-by-step recipes for common tasks.

---

## Add a New Page to the Site

### 1. Create the Sanity schema

Create `studio/schemaTypes/myPage.ts`:

```typescript
import { defineType, defineField } from 'sanity'

// Copy the localizedString/localizedBlockContent helpers from an existing schema

export const myPage = defineType({
  name: 'myPage',
  title: 'My Page',
  type: 'document',
  fields: [
    ...localizedString('title', 'Title', { required: true }),
    ...localizedString('subtitle', 'Subtitle'),
    ...localizedBlockContent('body', 'Body'),
  ],
  preview: {
    prepare() {
      return { title: 'My Page' }
    },
  },
})
```

### 2. Register the schema

In `studio/schemaTypes/index.ts`, import and add to the array:

```typescript
import { myPage } from './myPage'
export const schemaTypes = [...existing, myPage]
```

### 3. Add to Studio sidebar (if singleton)

In `studio/structure.ts`, add to `SINGLETONS` and add a sidebar item.

### 4. Deploy the Studio

```bash
cd studio
npm run deploy
```

### 5. Add Go model

In `site/models.go`:

```go
type MyPage struct {
    Title    string
    Subtitle string
    Body     string
}
```

### 6. Add Sanity fetch method

In `site/sanity.go`, add a `FetchMyPage(lang string)` method following the pattern of existing methods.

### 7. Add to cache

In `site/cache.go`:
- Add field to `contentCache` struct
- Add accessor method
- Add to `Refresh()` method

### 8. Add handler

In `site/handlers.go`:

```go
func handleMyPage(w http.ResponseWriter, r *http.Request) {
    lang := r.PathValue("lang")
    // ... read from cache, fall back to stub
    renderPage(w, r, "my-page", &PageData{...})
}
```

Add a stub function too.

### 9. Add route

In `site/routes.go`, add to the `switch` in `routeDispatcher`:

```go
case pagePath == "my-page":
    handleMyPage(w, r)
```

### 10. Add `PageData` field

In `site/render.go`, add the field to `PageData`:

```go
MyPage *MyPage
```

### 11. Create template

Create `site/templates/pages/my-page.html`:

```html
{{define "content"}}
<main>
  <h1>{{.MyPage.Title}}</h1>
  {{safeHTML .MyPage.Body}}
</main>
{{end}}
```

### 12. Update SEO

Add to sitemap, llms.txt, and llms-full.txt in `site/seo.go`.

### 13. Add to navigation

Update `site/templates/partials/navbar.html` and add locale strings to all `locales/*.json`.

---

## Add a New UI String (Translation)

1. Add key + English text to `site/locales/en.json`
2. Add translations to `es.json`, `de.json`, `nl.json`
3. Use in templates: `{{t .T "your_key"}}`

---

## Add a New Blog Post

1. Go to https://marina-alta-college.sanity.studio
2. Click **Journal** → **+** (create new)
3. Fill in title, slug (click Generate), excerpt, body, author, date
4. Click **Publish**

The post appears on the site within seconds.

---

## Add a Calendar Event

1. Go to the Studio → **Calendar** → **+**
2. Set title, event type, dates
3. Publish

---

## Update the CSS Theme

1. Edit `site/static/css/input.css`
2. Rebuild: `make css` or let `make dev` auto-rebuild
3. Commit and push to deploy

---

## Run the Site Without Sanity (Stub Mode)

Just don't set `SANITY_PROJECT_ID`:

```bash
# .env
PORT=8080
# SANITY_PROJECT_ID=  (commented out or empty)
```

The site serves built-in stub content for all pages.

---

## Set Up a Development Environment from Scratch

```bash
# 1. Install prerequisites
brew install go node

# 2. Clone repos
git clone git@github.com:marina-alta-college/site.git
git clone git@github.com:marina-alta-college/studio.git

# 3. Set up the site
cd site
npm install
cp .env.example .env
echo "SANITY_PROJECT_ID=e8pjc321" >> .env
make dev
# Site running at http://localhost:8080

# 4. Set up the Studio (optional, in another terminal)
cd ../studio
npm install
npm run dev
# Studio running at http://localhost:3333
```

---

## Force Refresh Content Cache

If content seems stale, trigger a webhook manually:

```bash
curl -X POST https://marinaaltacollege.com/webhook/sanity
```

Note: without a valid signature this will be rejected if `SANITY_WEBHOOK_SECRET` is set. For local development, just restart the server.

---

## Deploy the Site

Push to `main` on the `site` repo. Render auto-deploys.

```bash
cd site
git add .
git commit -m "your change"
git push origin main
```

---

## Deploy the Studio

```bash
cd studio
npm run deploy
```

---

## Change the Domain

1. Update `BASE_URL` env var on Render
2. Update DNS records (Render dashboard shows what to set)
3. Update CORS origins in Sanity dashboard if needed
4. Redeploy the site

---

## Add a New Language

See [i18n documentation](./i18n.md#adding-a-new-language) for the full checklist.

---

## Troubleshoot: Site Shows Stub Content in Production

1. Check Render logs for Sanity fetch errors
2. Verify `SANITY_PROJECT_ID` is set correctly in Render env vars
3. Check Sanity dashboard for API status
4. Try triggering a manual webhook (see above)
5. Check that content is published (not draft) in the Studio
