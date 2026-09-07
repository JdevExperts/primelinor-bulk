# PrimeLinor Bulk

## Overview

PrimeLinor Bulk is a full-stack B2B custom-products marketplace: a public
catalogue (apparel, corporate gifting, promotional products), a
Request-a-Quote intake flow, and a staff Admin panel that turns intake
into a sent, trackable quotation. There is no cart, checkout, or payment
flow — the commercial process ends at quote acceptance; fulfillment
happens outside the app. Production domain: **https://primelinorbulk.com/**.

## Architecture

Two independently deployable projects that only talk over HTTP — neither
has a filesystem dependency on the other.

- **Frontend** (`frontend/`) — React 19 + Vite, client-side routed (React
  Router). Built to static assets (`frontend/dist/`) and served by Nginx.
- **Backend** (`backend/`) — Node + Express 5 + Prisma ORM, a stateless
  JSON API under `/api/v1`. Runs as a long-lived Node process behind
  Nginx (`TRUST_PROXY=1`).
- **Database** — PostgreSQL, accessed exclusively through Prisma
  (`backend/prisma/schema.prisma` + `backend/prisma/migrations/`).
- **Object storage** — AWS S3 holds all managed catalogue images
  (products/categories/solutions) and customer-uploaded artwork. Local
  disk storage is a dev-only fallback and is refused outright at startup
  when `NODE_ENV=production`.
- **Reverse proxy** — Nginx terminates TLS and serves the built frontend
  at `/` on `primelinorbulk.com`/`admin.primelinorbulk.com`. The API is
  proxied on its **own subdomain**, `api.primelinorbulk.com` (whole host
  → the backend's internal port), not a path under the main domain. See
  [Nginx Routing](#nginx-routing).

## Repository Structure

```
/
  frontend/            React + Vite app (customer site + admin panel)
    src/
    public/
    .env.example
  backend/              Express + Prisma API
    src/
    prisma/             schema.prisma + migrations/
    scripts/            operational/maintenance scripts
    test/
    server.js
    DEPLOYMENT.md        deploy, backup/restore, prod bootstrap runbook
    .env.example
  package.json          root orchestration scripts (dev/build/lint/test)
  README.md             this file
  .gitignore
```

`backups/` (local Postgres dumps) and `node_modules/` exist on disk in
this checkout but are intentionally untracked — see
[Security Notes](#security-notes).

## Core Modules

- **Products / Categories** — `Product` ↔ `Category` is many-to-many
  (`ProductCategory`); each product has one `primaryCategoryId` plus
  optional secondary categories for cross-listing. Fully editable via
  Admin; no catalogue change requires a code deploy.
- **Solutions** — curated, named bundles of real products aimed at a
  specific buyer need (e.g. "Employee Welcome Kits"), each with an
  Admin-managed hero image and copy.
- **RFQ / Leads** — a customer's "Request a Quote" creates a `Lead` +
  `RFQ`, surfaced on the Admin Leads/RFQs inbox.
- **Quotations** — staff turn an RFQ into a `Quotation` (immutable once
  sent, versioned). The customer receives a `/quote/:token` link to
  view/download a PDF and Accept, Decline, or Request a Revision.
- **Corporate Gifting** — a dedicated marketing/catalogue surface
  (`/corporate-gifting`) for gift-kit-oriented buyers, built on the same
  Product/Solution data.
- **Analytics / Dashboard** — first-party, cookie-free website analytics
  (`AnalyticsEvent`: page views, product views, RFQ funnel, WhatsApp/
  contact clicks) ingested via `POST /api/v1/analytics/collect` and
  summarized on the Admin Dashboard (Overview, Website, Sales, Products,
  Catalogue Health tabs).
- **Product QA (`PRODUCT_REVIEW_PENDING`)** — a per-product attribute flag
  admins use to flag/track catalogue entries needing another look
  ("Reopen Review" sets it; "Mark Review Complete" removes it — its
  absence means review complete).
- **Studio** — an interactive customization workflow
  (`/customize/:productId`) for `customizable=true` products that have a
  `CUSTOMIZATION_FRONT` asset and an active `FRONT` placement zone
  (`studioReady`). Non-ready customizable products still work through the
  standard Request-a-Quote flow.

## Prerequisites

- Node.js ≥ 20
- npm
- PostgreSQL (local instance for development)
- AWS account with an S3 bucket + credentials (for artwork/image storage
  in any environment where you don't want the local-disk dev fallback)

## Local Development Setup

```bash
git clone git@github.com:JdevExperts/primelinor-bulk.git
cd primelinor-bulk

cd backend && npm install && cp .env.example .env   # fill in DATABASE_URL etc.
cd ../frontend && npm install
```

Or use the root orchestration scripts once both `.env` files exist:

```bash
npm run dev:backend    # terminal 1
npm run dev:frontend   # terminal 2
```

Frontend: http://localhost:5173/ · Backend: http://localhost:4001/

## Backend Environment Variables

Full annotated template: [`backend/.env.example`](./backend/.env.example).
Highlights (see the file for defaults and comments):

| Variable | Purpose |
|---|---|
| `PORT` | HTTP port the server listens on (default `4001`) |
| `DATABASE_URL` | PostgreSQL connection string — required in every environment |
| `FRONTEND_ORIGIN` | Comma-separated CORS allowlist |
| `PUBLIC_APP_URL` | Canonical frontend origin — builds `/quote/:token` links and the sitemap |
| `BACKEND_PUBLIC_URL` | This backend's own reachable base URL (local artwork preview links, dev only) |
| `JWT_SECRET` | Signs the admin session cookie — required in production |
| `ARTWORK_URL_SECRET` | Signs dev-only local artwork preview links (unused once S3 is configured) |
| `TRUST_PROXY` | Set `1` behind Nginx/ALB so `req.ip` is correct |
| `AWS_REGION` / `AWS_S3_BUCKET` / `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `S3_BASE_URL` | S3 object storage — all three of bucket/key/secret required in production |
| `WHATSAPP_NUMBER` | Digits + country code; blank disables the WhatsApp CTA |
| `SUPPORT_EMAIL` | Optional contact channel, shown only if set |
| `NODE_ENV` | `production` enables secure cookies, fail-fast config, script refusals |
| `ALLOW_DEV_SEED` | Dev/maintenance only — required to run `prisma/seed.js` when `NODE_ENV=production` |
| `ALLOW_ADMIN_BOOTSTRAP` | Dev/maintenance only — required to run `prisma/createStaffUser.js` when `NODE_ENV=production` |

Startup validation (`backend/src/startup/validateConfig.js`) refuses to
boot in production if `DATABASE_URL`, `JWT_SECRET`, `FRONTEND_ORIGIN`,
`PUBLIC_APP_URL`, or a complete S3 credential set is missing, and prints
exactly what's missing rather than starting half-configured.

## Frontend Environment Variables

Full template: [`frontend/.env.example`](./frontend/.env.example).

| Variable | Purpose |
|---|---|
| `VITE_API_BASE_URL` | Backend API base URL — **build-time**, baked into the bundle by `vite build` |
| `VITE_USE_MOCK_CATALOG` | Dev-only opt-in to serve the local mock catalogue instead of the real API; never active in a production build regardless of value |

No backend-only secret is ever exposed under a `VITE_` prefix.

## Database Setup

```bash
cd backend
createdb primelinor_dev            # or your provider's equivalent
cp .env.example .env               # set DATABASE_URL to point at it
npm run prisma:migrate             # dev workflow — creates tables
npm run seed                       # optional, dev-only demo catalogue
```

## Prisma Workflow

- **Local dev**: `npm run prisma:migrate` (`prisma migrate dev`) — may
  prompt for/apply destructive changes; dev database only.
- **Every other environment**: `npm run prisma:deploy` (`prisma migrate
  deploy`) — applies committed migrations only, never destructive,
  never interactive.
- Schema source of truth: `backend/prisma/schema.prisma`. Every schema
  change ships as a new migration under `backend/prisma/migrations/`,
  committed to git — migration SQL is never gitignored.

## Running Locally

```bash
npm run dev:backend     # nodemon server.js
npm run dev:frontend    # vite dev server
```

## Building Frontend

```bash
cd frontend
npm ci
npm run lint     # oxlint
npm run build     # → frontend/dist/
```

Set `VITE_API_BASE_URL` in the build environment **before** running
`npm run build` — it is compiled into the static bundle, not read at
runtime.

## Backend Production Start

```bash
cd backend
npm ci --omit=dev
npm run prisma:generate
npm run prisma:deploy
npm start          # node server.js
```

Run under a process manager or container orchestrator that restarts on
crash (pm2/systemd/container restart policy — none is bundled; pick
whichever fits the actual host). Health check: `GET /health` — `200
{"status":"ok"}` only once a real `SELECT 1` against the database
succeeds, `503` otherwise; point the process manager / load balancer's
health probe here.

## Production Deployment Overview

See [`backend/DEPLOYMENT.md`](./backend/DEPLOYMENT.md) for the full
pre-deploy checklist, required env vars, and the production
seed/bootstrap plan. Summary:

1. Build the frontend with the production `VITE_API_BASE_URL` set.
2. Install backend deps, `prisma generate`, `prisma migrate deploy`.
3. Set every required production env var (table above) — the server
   fails fast at boot if anything's missing.
4. Start the backend under a process manager, pointed at by Nginx.
5. Deploy `frontend/dist/` where Nginx serves it.
6. Create the first admin account (§ below) before advertising the admin
   UI as live.

## Nginx Routing

**Verified against the live production host (confirmed via direct
inspection, September 2026) — the API lives on its own subdomain, not a
path under the main domain.** `primelinorbulk.com` and
`admin.primelinorbulk.com` share one static root and have **no** `/api/`
location block; `api.primelinorbulk.com` is a fully separate server block
that proxies its entire host to the backend.

```nginx
# ── API — its own subdomain, whole host proxied to the backend ─────────────
server {
    server_name api.primelinorbulk.com;

    location / {
        proxy_pass http://localhost:4001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        client_max_body_size 20M;
    }

    listen 443 ssl; # managed by Certbot
}

# ── Main website — static SPA, no API location block ────────────────────────
server {
    server_name primelinorbulk.com www.primelinorbulk.com;

    root /var/www/primelinorbulk;
    index index.html;

    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # SPA fallback — required for client-side routing (React Router)
    location / {
        try_files $uri $uri/ /index.html;
    }

    # /admin is served from its own subdomain, not this one
    location /admin {
        return 301 https://admin.primelinorbulk.com$request_uri;
    }

    listen 443 ssl; # managed by Certbot
}

# ── Admin — same static root as the main site, different entry behavior ────
server {
    server_name admin.primelinorbulk.com;

    root /var/www/primelinorbulk;
    index index.html;

    location = / {
        return 301 /admin/login;
    }

    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    location / {
        try_files $uri $uri/ /index.html;
    }

    listen 443 ssl; # managed by Certbot
}
```

This documents the actual live config for reference — no Nginx config is
modified by this repository; the live host is managed directly.

## Port Mapping

| Environment | Component | Port / URL | Purpose |
|---|---|---|---|
| Local | Frontend (Vite dev server) | `http://localhost:5173` | Frontend dev |
| Local | Backend (Express) | `http://localhost:4001` | Backend dev/API |
| Production | Public frontend | `https://primelinorbulk.com/`, `https://www.primelinorbulk.com/` | Nginx serves `/var/www/primelinorbulk` (built `frontend/dist/`) |
| Production | Admin panel | `https://admin.primelinorbulk.com/` | Same static root as the main site; `/` redirects to `/admin/login` |
| Production | Public API | `https://api.primelinorbulk.com/api/v1/` | **Its own subdomain** — not a path under `primelinorbulk.com` |
| Production | Backend internal | `127.0.0.1:4001` | Node process (PM2: `primelinorbulk-api`), not directly internet-facing |

`4001` is the backend's built-in default (`process.env.PORT || 4001` in
`server.js`) and matches the previous `primelinor-bulk` codebase's own
fallback, confirmed identical to the live production PM2 process — no
change needed. **`VITE_API_BASE_URL` must be
`https://api.primelinorbulk.com/api/v1` in production** (the subdomain,
not `https://primelinorbulk.com/api/v1`) — the main domain's Nginx block
has no `/api/` location, so a build pointed at the wrong host silently
serves `index.html` instead of JSON for every API call. This exact
mistake happened once during the live cutover and was caught by checking
real network requests in a browser, not by curling the API host directly
(curling `api.primelinorbulk.com` in isolation looks fine even when the
frontend bundle points somewhere else) — rebuild and redeploy
(`npm run build` + copy `dist/` to `/var/www/primelinorbulk`) if this
value is ever wrong.

## Production Database Promotion

There is no seeded default production catalogue. The current local
development database is the production-ready catalogue source (Products,
Categories, Solutions have been through completeness/cleanup passes) —
promote it via `pg_dump`/`pg_restore` rather than re-deriving it. See
`backend/DEPLOYMENT.md` §4 for the full plan, including the S3 bucket
remapping consideration if production uses a different bucket than the
promoted dev data was pointing at.

## Backups / Restore

Dumps live outside git (`backups/`, gitignored — see
[Security Notes](#security-notes)). Restoring a verified dump such as
`backups/primelinor-bulk-preprod-20260904.dump` into a fresh production
database:

```bash
createdb primelinor_production
pg_restore --no-owner --no-privileges -d primelinor_production \
  backups/primelinor-bulk-preprod-20260904.dump
cd backend && npm run prisma:deploy   # applies any migration newer than the dump
```

Full backup/restore runbook, retention policy, and the pre-migration
backup discipline: `backend/DEPLOYMENT.md` §3.

## S3

Non-secret production config (never commit credentials):

```
AWS_REGION=ap-south-1
AWS_S3_BUCKET=pl-bulk
S3_BASE_URL=https://pl-bulk.s3.ap-south-1.amazonaws.com
```

The bucket needs a public-read policy on the `products/*`, `categories/*`,
`solutions/*`, and `images/*` (legacy import) key prefixes — see
`backend/README.md`'s S3 bucket policy section for the exact policy JSON.
Object uploads never set a per-object ACL; the bucket's own policy is what
makes them publicly readable.

## Admin Users / Bootstrap Safety

There is no seeded default admin — `prisma/seed.js` never creates a
`StaffUser` row, to avoid a predictable baked-in credential. Create the
first admin once, after `prisma:deploy` succeeds and before the admin UI
is advertised as live:

```bash
NODE_ENV=production ALLOW_ADMIN_BOOTSTRAP=true \
  node prisma/createStaffUser.js --email=you@company.com --name="Your Name" --role=ADMIN
```

Omit `--password` to have the script generate and print a strong random
one exactly once — capture it immediately; it is never logged again.
`ALLOW_ADMIN_BOOTSTRAP` and `ALLOW_DEV_SEED` are maintenance-only escape
hatches — never set them as a persisted var in the long-running
production process.

## Analytics

First-party, cookie-free page/event analytics
(`backend/prisma/migrations/20260903120000_add_analytics_events`):
`PAGE_VIEW`, `PRODUCT_VIEW`, `PRODUCT_CARD_CLICK`, `CATEGORY_VIEW`,
`SOLUTION_VIEW`, `SEARCH`, `QUOTE_CTA_CLICK`, `RFQ_STARTED`,
`RFQ_SUBMITTED`, `WHATSAPP_CLICK`, `CONTACT_CLICK` — ingested via `POST
/api/v1/analytics/collect` (its own rate limiter) and summarized on the
Admin Dashboard's Website/Sales/Products/Catalogue Health tabs.

## Product Review Workflow

`PRODUCT_REVIEW_PENDING` is a per-product attribute (`ProductAttribute`)
admins use to flag a catalogue entry needing another look. Admin's
"Reopen Review" action sets it (`key=PRODUCT_REVIEW_PENDING,
value=true`); "Mark Review Complete" removes the row entirely — its
absence means review complete, not a `false` value.

## Security Notes

- `.env` / `.env.*` are gitignored everywhere except the checked-in
  `**/.env.example` templates — no real secret has ever been tracked.
- Database dumps (`backups/`, `*.dump`, `*.sql.gz`, `*.backup`) are
  gitignored and never committed — they contain customer PII (Lead/RFQ
  contact details, quotation recipients).
- `backend/prisma/migrations/**/migration.sql` is **never** gitignored —
  the full migration history stays tracked.
- **Legacy history warning**: an earlier commit in this repository's
  history (before the current `.gitignore` rules) contains a legacy DB
  dump with real password hashes and customer PII. It was removed from
  the working tree but remains reachable in git history; a full history
  rewrite (`git filter-repo`/BFG) is a separate, deliberate action for
  the repository owner — not performed as part of this audit.
- Production S3 credentials, `JWT_SECRET`, and `ARTWORK_URL_SECRET` must
  be real, random, per-environment values — never the `.env.example`
  placeholders.

## Testing

```bash
cd backend && npm test     # node --test, 430 tests
```

No frontend test suite exists today (`frontend/package.json` has no
`test` script) — lint and build are the frontend's automated checks.

## Lint / Build

```bash
npm run lint:frontend      # oxlint — 0 errors
npm run build:frontend     # vite build
npm run test:backend       # node --test
npm run verify              # all three, in order
```

## Troubleshooting

- **Server refuses to boot in production**: read the printed error list
  from `src/startup/validateConfig.js` — it names every missing required
  var in one pass rather than failing on the first one.
- **Artwork/image upload "succeeds" then 404s later**: almost always a
  missing S3 bucket-policy statement for that asset type's key prefix
  (see [S3](#s3)) — production refuses local-disk fallback outright, so
  this is a policy gap, not a code bug.
- **`/health` returns 503**: database is unreachable from the backend
  process — check `DATABASE_URL` and network/security-group access, not
  application code.
- **Shared quote link (`/quote/:token`) is wrong/broken**: check
  `PUBLIC_APP_URL` — it, not `FRONTEND_ORIGIN`, is what builds that link
  and the sitemap.

## Rollback Overview

1. Keep the previous backend process/version available (don't remove the
   old deployment artifact until the new one is verified).
2. If a migration shipped with the deploy, restore the pre-migration
   `pg_dump` backup taken before deploying (see
   [Backups / Restore](#backups--restore)) rather than attempting to hand
   -write a down-migration.
3. Point Nginx's `proxy_pass`/document root back at the previous
   backend/frontend build, `nginx -t`, reload.
4. Verify `/health` and a few real pages/API calls before declaring the
   rollback complete.
5. Keep the failed deploy's logs and the backup used for rollback until
   the incident is understood — don't reuse that database state for a
   fresh migration attempt without confirming it's actually consistent.
