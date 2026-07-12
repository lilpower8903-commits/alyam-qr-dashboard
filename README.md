# ALYAM Command Center

Production-ready QR tracking, branch operations, analytics, reviews intelligence, AI Manager, and executive reporting for ALYAM. The application is built with Next.js App Router and deploys to **Cloudflare Workers** through OpenNext, using **Cloudflare D1** for relational data and **Cloudflare KV** for live-event signaling and edge rate limiting.

## Cloudflare architecture

| Layer | Cloudflare service | Responsibility |
| --- | --- | --- |
| Application | Workers + OpenNext | Next.js UI, API routes, QR redirects, AI Manager, reports |
| Database | D1 | Branches, scans, reviews, notifications, integrations, audit events |
| Edge state | KV | QR rate-limit windows and cross-isolate live-event signal |
| Static delivery | Workers Assets | Next.js static assets and PWA assets |
| Secrets | Workers Secrets | Session HMAC secret, IP hash secret, optional Google credentials |

The runtime uses only Cloudflare D1 and KV bindings. OpenNext runs Next route handlers in its Cloudflare Workers-compatible `nodejs` route runtime, but application code uses no Node-only APIs: there is no external database URL, no connection pool, and no Node crypto, `Buffer`, or `process` usage. Password hashing, session HMAC, and IP hashing use Workers Web Crypto. Every D1 write uses prepared statements; QR scans use a D1 batch for the event and counter update.

## Capabilities

- Unique branded QR codes per branch; browser-generated **PNG, SVG, and PDF** downloads
- Fast `/scan/{branch-code}` flow: validate, capture analytics, increment counter, HTTP 302 redirect
- D1 scan analytics: date/time, device, browser, edge country/city when available, destination type, privacy-safe IP hash
- D1-backed branches CRUD, active state, Maps/Review/WhatsApp URLs, QR color, coordinates, contacts, source metadata
- All-branches command center, branch detail analytics, reviews, keywords, sentiment categories, peak hours, QR lifecycle
- Live dashboard refresh signal via KV-backed Server-Sent Events
- Executive metrics, rankings, traffic trends, device/browser/country breakdowns, daily Excel/CSV and executive PDF reports
- AI Manager answers in Arabic or English from real stored system data only
- Google Business Profile integration layer; credentials stay in Worker secrets
- PWA manifest + service worker; API and QR scan paths are deliberately excluded from offline cache
- Open workspace by default; set `AUTH_MODE=required` to use signed admin sessions
- Web Crypto PBKDF2 password hashes, signed HTTP-only cookies, input validation, D1 prepared statements, KV edge rate limiting
- Self-healing operational checks for inactive QR branches, 48-hour scan silence, and ratings below 4.0

## Repository layout

```text
src/app/                 Next.js pages and Worker-compatible route handlers
src/components/          Dashboard and QR UI
src/db/                  D1 Drizzle schema and request-scoped D1 adapter
src/lib/                 Analytics, D1 utilities, Web Crypto, KV, AI, integrations
migrations/              Generated SQLite/D1 migrations
public/                  PWA manifest, service worker, icons
wrangler.jsonc           Worker, D1, KV, and asset binding configuration
open-next.config.ts      OpenNext Cloudflare runtime configuration
.dev.vars.example        Local Worker secret template
```

## Prerequisites

- Node.js 20 or newer
- A Cloudflare account
- Wrangler authentication: `npx wrangler login`

## First-time Cloudflare setup

### 1. Install dependencies

```bash
npm ci
```

### 2. Create the D1 database

```bash
npx wrangler d1 create alyam-command-center-db
```

Copy the returned `database_id` into `wrangler.jsonc` at:

```jsonc
"d1_databases": [{ "binding": "ALYAM_DB", "database_id": "..." }]
```

### 3. Create the KV namespace

```bash
npx wrangler kv namespace create ALYAM_KV
```

Copy the returned namespace `id` into `wrangler.jsonc` at:

```jsonc
"kv_namespaces": [{ "binding": "ALYAM_KV", "id": "..." }]
```

### 4. Generate Worker binding types

```bash
npm run cf-typegen
```

### 5. Configure local secrets

```bash
cp .dev.vars.example .dev.vars
```

Generate strong values locally:

```bash
openssl rand -base64 48
```

Set `AUTH_SECRET` and `IP_HASH_SECRET` in `.dev.vars`. Do not commit `.dev.vars`.

### 6. Apply the D1 schema locally

```bash
npm run db:migrate:local
```

The complete D1 schema is in `migrations/0000_flaky_silk_fever.sql`.

### 7. Run the Worker locally

```bash
npm run preview
```

This runs the OpenNext transformation and then Wrangler preview with local D1/KV bindings. On a new database, the first branch/dashboard request imports the live ALYAM branch directory. The importer stores only data returned by ALYAM public sources.

## Production deployment

### 1. Store Worker secrets

```bash
npx wrangler secret put AUTH_SECRET
npx wrangler secret put IP_HASH_SECRET
```

Optional Google Business Profile secrets:

```bash
npx wrangler secret put GOOGLE_BUSINESS_ACCESS_TOKEN
npx wrangler secret put GOOGLE_BUSINESS_ACCOUNT_ID
```

Set `AUTH_MODE` to `required` in `wrangler.jsonc` only if you want to require the existing signed admin session flow. The shipped configuration uses `open` mode as requested for the operational workspace.

### 2. Apply D1 migrations remotely

```bash
npm run db:migrate:remote
```

### 3. Validate and deploy

```bash
npm test
npm run cf-typegen
npm exec tsc -- --noEmit --pretty false
npm run build
npm run deploy
```

`npm run deploy` creates the OpenNext Worker bundle and deploys it through Wrangler. Cloudflare returns the `workers.dev` URL. Configure a custom domain from **Workers & Pages → alyam-command-center → Settings → Domains & Routes**.

## QR test checklist

Before launch, test using the deployed Worker URL or assigned custom domain:

```text
https://your-domain.example/scan/{branch-code}
```

- Generate each QR as PNG, SVG, and PDF from the branch card
- Scan with iPhone and Android camera applications
- Confirm a single HTTP 302 response opens the configured Google Maps/Review destination
- Confirm one new event is shown in **Scan activity** and the branch counter increments once
- Check device/browser/country/city fields when Cloudflare edge headers are available
- Verify the daily Excel report contains branch summary, current-day scans, and review signals
- Test desktop, tablet, Chrome, Safari, and Edge layouts

## Database migration workflow

Schema is defined in `src/db/schema.ts` using `drizzle-orm/sqlite-core`.

```bash
# Generate a new migration after a schema change
npx drizzle-kit generate --config=drizzle.config.ts

# Test it against local D1 first
npm run db:migrate:local

# Then apply it to production D1
npm run db:migrate:remote
```

Use only the D1 migration workflow above; the project is D1-only.

## D1 backup and recovery

Export data through Cloudflare:

```bash
npx wrangler d1 export alyam-command-center-db --remote --output=alyam-d1-backup.sql
```

For operational reporting, use the dashboard’s **Excel**, **CSV**, and **Executive PDF** downloads. To restore a SQL backup, review it first and execute it against a replacement D1 database using Wrangler’s D1 execute command.

## Google Business Profile integration

1. Create a Google OAuth access token with review read permission.
2. Store it with `wrangler secret put GOOGLE_BUSINESS_ACCESS_TOKEN`.
3. Store the Google Business account ID with `wrangler secret put GOOGLE_BUSINESS_ACCOUNT_ID`.
4. Set the relevant `googleLocationId` on each branch as part of the secure operations workflow.
5. Use **Integrations → Sync now**.

The Worker calls Google server-side, stores ratings/reviews in D1, performs local sentiment/category analysis, and creates follow-up notifications. No secret is sent to the browser.

## Free plan considerations

- D1 and KV usage is bounded by Cloudflare Free plan quotas; inspect Cloudflare dashboard metrics before high-volume rollout.
- The public QR limiter is best-effort KV fixed-window protection. Pair it with Cloudflare WAF/rate-limit rules if available on your zone for stronger perimeter defense.
- The OpenNext configuration uses static-assets incremental cache to avoid adding paid storage dependencies.
- D1 is strongly consistent at the primary. Dashboard live notifications use KV as a low-cost, eventually consistent signal; the dashboard always refreshes actual counters from D1.

## Quality gates

```bash
npm test
npx next typegen
npm exec tsc -- --noEmit --pretty false
npm run build
npx opennextjs-cloudflare build
```

## Security notes

- All branch input is validated with Zod before D1 writes.
- D1 operations use Drizzle or prepared binding values; no interpolated user SQL is used.
- Scan visitor IP addresses are salted and SHA-256 hashed before storage; plaintext IPs are never persisted.
- Session cookies are HTTP-only, secure, and same-site lax.
- `AUTH_MODE=required` enables authenticated admin routes; the current `open` default intentionally preserves the requested no-login workspace behavior.
- Keep `AUTH_SECRET`, `IP_HASH_SECRET`, and Google credentials as Cloudflare secrets only. Never add them to `NEXT_PUBLIC_*` variables.
