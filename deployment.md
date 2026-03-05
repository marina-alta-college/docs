# Deployment & Hosting

## Production Site (Render)

The site is deployed as a **Go web service** on [Render](https://render.com) in the **Frankfurt** region.

### Render Configuration

Defined in `site/render.yaml`:

```yaml
services:
  - type: web
    name: marina-alta-college
    runtime: go
    region: frankfurt
    plan: starter
    buildCommand: npm install && npx @tailwindcss/cli -i static/css/input.css -o static/css/output.css --minify && go build -o site .
    startCommand: ./site
    envVars:
      - key: PORT
        value: 10000
      - key: SANITY_PROJECT_ID
        value: e8pjc321
      - key: SANITY_DATASET
        value: production
      - key: SANITY_API_VERSION
        value: 2025-03-03
```

### Environment Variables on Render

These must be set in the Render dashboard (not all are in `render.yaml`):

| Variable | Value | Notes |
|---|---|---|
| `PORT` | `10000` | Render's default |
| `SANITY_PROJECT_ID` | `e8pjc321` | |
| `SANITY_DATASET` | `production` | |
| `SANITY_API_VERSION` | `2025-03-03` | |
| `SANITY_WEBHOOK_SECRET` | (secret) | Must match Sanity webhook config |
| `BASE_URL` | `https://marinaaltacollege.com` | Used for sitemaps and SEO |

### How Deploys Work

1. Push to `main` branch on GitHub
2. Render detects the push and starts a build
3. Build: `npm install` → Tailwind CSS build → `go build`
4. Render replaces the running service with the new binary
5. On startup: load translations → parse templates → warm cache from Sanity

### Manual Deploy

In the Render dashboard, click **Manual Deploy** → **Deploy latest commit**.

### Custom Domain

Configure in Render dashboard under the service → **Settings** → **Custom Domains**:
- Add `marinaaltacollege.com`
- Add `www.marinaaltacollege.com` (redirect to apex)
- Update DNS records as Render instructs

## Sanity Studio

The Studio is deployed to Sanity's hosting:

```bash
cd studio
npm run deploy
```

This deploys to https://marina-alta-college.sanity.studio. The `studioHost` is set in `sanity.cli.ts`.

### Sanity Project Management

- Dashboard: https://www.sanity.io/manage/project/e8pjc321
- Here you can manage: API tokens, CORS origins, webhooks, datasets, team members

### CORS Origins

The Sanity project needs CORS origins for:
- `http://localhost:3333` (local Studio development)
- `https://marina-alta-college.sanity.studio` (deployed Studio)

These are configured in the Sanity dashboard under API → CORS Origins.

## Webhook Setup

To ensure content changes appear on the live site:

1. Go to Sanity dashboard → API → Webhooks
2. Create a webhook:
   - **Name**: Site cache refresh
   - **URL**: `https://marinaaltacollege.com/webhook/sanity`
   - **Trigger on**: Create, Update, Delete
   - **Filter**: (leave empty — refresh on any change)
   - **Secret**: Generate a strong secret, then set the same value as `SANITY_WEBHOOK_SECRET` on Render

## Transferring Ownership

### GitHub Org

To transfer the `marina-alta-college` GitHub org:

1. Go to https://github.com/organizations/marina-alta-college/settings
2. Under **Danger zone** → **Transfer ownership**
3. The new owner accepts the transfer
4. Add the previous owner as a member/collaborator if continued access is needed

### Sanity Project

1. Go to https://www.sanity.io/manage/project/e8pjc321 → **Members**
2. Invite the new owner with **Administrator** role
3. They can then manage billing, team members, and API tokens

### Render

1. Go to Render dashboard → **Team Settings**
2. Invite the new owner
3. Transfer the service to their account if needed

### Domain

Transfer the domain registrar account or update nameservers as needed.
