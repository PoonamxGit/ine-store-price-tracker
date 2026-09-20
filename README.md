# INE Product Price Tracker

A price tracker built with React, Express, Supabase and Playwright for the INE demo store. It records product price and stock history, logs scrape attempts, and supports scheduled scraping through cron-job.org.

## Live Demo

- [Frontend](https://ine-store-price-tracker.vercel.app)
- [Backend](https://ine-store-price-tracker-api.onrender.com) · [Backend health](https://ine-store-price-tracker-api.onrender.com/api/health)
- [GitHub repository](https://github.com/PoonamxGit/ine-store-price-tracker)
- Recording: [Watch the 2–4 minute demo](https://drive.google.com/file/d/1qSUK4ls8ykpP9V6mQMPxJbSTe8nsTLtL/view?usp=sharing)

## What It Does

- Search by full or partial product name and track products from the store.
- Scrape real prices and stock, then view validated history, charts and per-product logs.
- Retry temporary failures and schedule scraping every two hours through cron-job.org.

A **FAILED** badge means the latest scrape attempt failed. Previously valid price and stock data remains available.

## Architecture

```text
Browser / React / Vercel -> Express API / Render -> Supabase PostgreSQL
                                   |
                            Playwright -> INE demo store
cron-job.org -> Render cron endpoint
```

React provides search, history and logs. Express handles validation and scraping, Supabase stores observations and jobs, and Playwright reads the store's dynamically produced price/stock. cron-job.org triggers asynchronous work through the API.

## Reliability

The scraper checks product identity and validates price/stock before saving. Bounded retries log `success`, `retried` or `failed` attempts; failed attempts create no invalid `price_history` rows and leave previous successful data available. Database guards prevent concurrent scrapes of the same product, while a durable cron queue and two-hour slot deduplication prevent duplicate scheduled work.

## Project Structure

```text
client/             React UI, charts and API client
server/             Express API, scraper, worker and tests
supabase/           Initial PostgreSQL schema, RLS and functions
docs/               Design, recon, corrections and demo guides
Dockerfile          Backend with Chromium and OS dependencies
render.yaml         Render backend blueprint
vercel.json         Frontend build and SPA routing
.github/workflows/  GitHub Actions checks
```

## Quick Start

Install Node.js 22.12+ and npm. Run commands from the repository root.

```sh
git clone https://github.com/PoonamxGit/ine-store-price-tracker.git
cd ine-store-price-tracker
npm ci
npx playwright install chromium
```

On Linux, use `npx playwright install --with-deps chromium` to include browser system dependencies.

| Shell | Create root environment | Create client environment |
| --- | --- | --- |
| Windows PowerShell | `Copy-Item .env.example .env` | `Copy-Item client/.env.example client/.env` |
| macOS/Linux | `cp .env.example .env` | `cp client/.env.example client/.env` |

Edit root `.env` with your settings:

| Variable | Purpose / local value |
| --- | --- |
| `PORT` | API listening port; `3001` |
| `NODE_ENV` | `development` locally, `production` on Render |
| `SUPABASE_URL` | Your Supabase project API URL |
| `SUPABASE_SERVICE_ROLE_KEY` | Server-side Supabase secret/service-role key |
| `FRONTEND_URL` | Allowed frontend origin; `http://localhost:5173` |
| `CRON_SECRET` | Random server-only secret, at least 32 characters |
| `SCRAPER_TIMEOUT_MS` | Per-attempt deadline; `45000` |
| `SCRAPER_MAX_ATTEMPTS` | Total attempts; `3` (maximum 3) |

Set `VITE_API_BASE_URL=http://localhost:3001` in `client/.env`. `SUPABASE_SERVICE_ROLE_KEY` and `CRON_SECRET` are server-only secrets: never commit `.env` files or put server secrets in Vercel frontend variables.

## Database Setup

1. Create a Supabase project and open **SQL Editor**.
2. Run the full [supabase/schema.sql](supabase/schema.sql) against a fresh project; this is an initial schema, not a repeatable migration.
3. Add the Supabase URL and server-side secret to root `.env`.

Main tables: `tracked_products`, `scrape_runs`, `scrape_attempts`, `price_history`, and `cron_jobs`. RLS is enabled; application database access happens server-side.

## Run Locally

Run these commands from the repository root in separate terminals:

```sh
npm run dev:server  # Terminal 1
npm run dev:client  # Terminal 2
```

Open the [frontend](http://localhost:5173) and [backend health endpoint](http://localhost:3001/api/health). Health reports process liveness, not database connectivity.

## Main API Endpoints

| Method | Route | Purpose |
| --- | --- | --- |
| GET | `/api/health` | Process health |
| GET | `/api/store/search?q=Smartwatch` | Full/partial-name search and catalog coverage |
| GET | `/api/products` | Active tracked products |
| POST | `/api/products` | Track and initially scrape; JSON `{"sourceProductId":"763"}` |
| GET | `/api/products/:id` | Product details; `:id` is the tracked UUID |
| DELETE | `/api/products/:id` | Stop tracking; preserve history |
| GET | `/api/products/:id/history` | Paginated observations |
| GET | `/api/products/:id/logs` | Paginated scrape attempts |
| POST | `/api/products/:id/scrape` | Manual scrape |
| POST | `/api/cron/scrape` | Queue scheduled scraping |
| GET | `/api/cron/jobs/:id` | Inspect queued job status/results |

Both cron routes require `Authorization: Bearer <CRON_SECRET>`; missing/invalid secrets return HTTP 401. HTTP 202 acknowledges queued work, not successful extraction.

## Headed Scraper Demo

```sh
npm run scrape:headed -- --product=763
npm run scrape:headed -- --product=763 --fault=fail-first
npm run scrape:headed -- --product=763 --fault=slow
```

These commands save to your configured Supabase project. In development headed mode, `fail-first` deliberately aborts the first navigation and logs it as retried; a later successful attempt creates one valid observation. This is an injected demonstration failure, not a natural store failure. `slow` delays metadata by five seconds. Fault injection is development/headed-only and rejected in production.

## Tests

```sh
npm run lint
npm test
npm run build
npm run test:browser
```

GitHub Actions runs dependency installation, lint, tests, production build, Playwright Chromium installation, and browser tests. Offline tests use isolated PostgreSQL/fixtures; browser tests need Chromium and free port 4175. Optional live-check scripts are listed in [package.json](package.json).

## Deployment

### Render backend

Create a Web Service from this repository, or use the [Render Blueprint](render.yaml).

| Setting | Value |
| --- | --- |
| Branch | `main` |
| Runtime | Docker |
| Root Directory | Leave blank (repository root) |
| Dockerfile | `./Dockerfile` |
| Health Check Path | `/api/health` |

Docker installs Chromium and its system dependencies, then starts the API. Set these environment variables:

```dotenv
NODE_ENV=production
PORT=3001
SUPABASE_URL=<your Supabase URL>
SUPABASE_SERVICE_ROLE_KEY=<server-only secret>
FRONTEND_URL=https://ine-store-price-tracker.vercel.app
CRON_SECRET=<32+ character random secret>
SCRAPER_TIMEOUT_MS=45000
SCRAPER_MAX_ATTEMPTS=3
```

`FRONTEND_URL` must exactly match your Vercel production origin, without a trailing slash. The server binds to `0.0.0.0` using the configured `PORT`.

### Vercel frontend

Import the same repository with these settings, matching [vercel.json](vercel.json):

| Setting | Value |
| --- | --- |
| Framework | Vite |
| Root Directory | Repository root |
| Install Command | `npm ci` |
| Build Command | `npm run build` |
| Output Directory | **`client/dist`** |

Set `VITE_API_BASE_URL=https://YOUR_RENDER_SERVICE.onrender.com`; this project uses `https://ine-store-price-tracker-api.onrender.com`. This is public frontend configuration, not a server secret. Redeploy after changing Vite variables because they are injected at build time.

## External Cron

Create a job in cron-job.org:

| Setting | Value |
| --- | --- |
| Title | INE Price Tracker - Every 2 Hours |
| Method | POST |
| URL | `https://YOUR_RENDER_SERVICE.onrender.com/api/cron/scrape` |
| Header | `Authorization: Bearer <CRON_SECRET>` |
| Body | Empty |
| Schedule | `0 */2 * * *` |
| Timezone | UTC |

Replace the hostname with your Render service. HTTP **202** means scraping is queued asynchronously; inspect the returned job ID and product logs for outcomes.

Optional free-tier wake-up job: **GET** `https://YOUR_RENDER_SERVICE.onrender.com/api/health`, schedule **`50 1-23/2 * * *`**, timezone **UTC** (ten minutes before each scrape). This is only a free-tier cold-start workaround, not part of scraping logic or an availability guarantee.

## Verification Checklist

- [ ] Health returns `{"status":"ok"}`; frontend loads; search and product tracking work.
- [ ] Valid price/stock appears; a manual scrape updates history/logs.
- [ ] Headed fail-first shows RETRIED then SUCCESS; failed attempts insert no invalid price history.
- [ ] Cron test returns HTTP 202; inspect job results and observe an actual scheduled cycle.
- [ ] GitHub Actions passes.

## Documentation

- [DESIGN.md](docs/DESIGN.md): Architecture, reliability decisions and trade-offs.
- [SCRAPER_RECON.md](docs/SCRAPER_RECON.md): Observed store behavior and extraction signals.
- [AI_NOTES.md](docs/AI_NOTES.md): Real implementation assumptions that were wrong and how they were corrected, as required by the assignment.
- [VERIFICATION.md](docs/VERIFICATION.md): Dated local test evidence and steps for verifying the hosted workflow.
- [VIDEO_GUIDE.md](docs/VIDEO_GUIDE.md): Recording sequence and headed demonstration steps.

## Known Operational Notes

[Render free instances](https://render.com/docs/free) may sleep and cold-start. Catalog discovery can take time because the demo catalog reshuffles and rate-limits requests. Only validated, completed observations enter price history; FAILED means the latest scrape failed, not that previous valid data was lost.

## Recording

Screen recording: [Watch the 2–4 minute demo](https://drive.google.com/file/d/1qSUK4ls8ykpP9V6mQMPxJbSTe8nsTLtL/view?usp=sharing)

The recording should demonstrate the live app, history/logs, a headed browser run, retry/failure handling, and cron setup.

## Security

`.env` files are ignored; server secrets stay backend-only. Cron uses bearer authentication. Supabase RLS and revoked grants block direct anonymous/authenticated table access. Do not expose `SUPABASE_SERVICE_ROLE_KEY` or `CRON_SECRET`. This is a shared assignment dashboard; CORS is not user authentication.

This repository was created as the INE Software Engineer Intern hands-on assignment.
