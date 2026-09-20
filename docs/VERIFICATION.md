# Verification Record

**Development date:** 2026-09-19 to 2026-09-20  
**Local environment:** Windows, Node.js 22, Playwright Chromium

This document records what was verified locally and in the hosted deployment. Test fixtures are used only for automated tests; the deployed application uses Supabase and the live INE demo store.

## Local Verification

| Check | Result / Evidence |
| --- | --- |
| INE store inspection | Verified the React storefront, catalog/product metadata, browser interaction flow, and visible price/stock behavior |
| Partial/full-name search | Live search logic verified against real catalog records |
| Real price and stock extraction | Verified with real INE products through the production scraper |
| Headed browser mode | Visible Chromium opens and performs the real product interaction |
| `fail-first` demonstration | First navigation is intentionally aborted, logged as `RETRIED`, then a later attempt succeeds |
| Slow-response demonstration | Development-only delay mode exercises slow handling without inventing product data |
| Retry/failure persistence | Retries and terminal failures are recorded; failed attempts do not create price-history observations |
| Validation | Product identity, price format, currency, stock status/quantity, and page structure are validated before persistence |
| SQL integrity | Schema/RPC tests cover atomic observation writes, duplicate prevention, run state, leases, and queue behavior |
| Concurrency/recovery | Per-product/global admission, cron deduplication, lease recovery, and interrupted-run handling are covered |
| Frontend | Search, tracking, history, chart, scrape logs, manual scrape, failure states, and responsive layout are covered |
| Build/lint | `npm run lint`, `npm test`, `npm run build`, and `npm run test:browser` pass locally |

## Hosted Deployment Verification

### Supabase

- A real Supabase PostgreSQL project is configured.
- `supabase/schema.sql` has been applied.
- The deployed Render backend connects to Supabase with server-only credentials.
- Tracked products, scrape runs, attempts, price history, and cron jobs are visible through the application.
- Previous valid observations remain available when a later scrape fails.

### Render

**Production backend:**  
https://ine-store-price-tracker-api.onrender.com

Verified:

- Docker deployment is live.
- `/api/health` returns:

```json
{"status":"ok"}
```

- The service uses the current GitHub repository.
- Supabase connectivity is working.
- CORS is configured for the production Vercel origin.
- Manual scrape requests reach the backend successfully.
- The authenticated cron endpoint accepts valid enqueue requests.
- Render free-tier cold starts are mitigated with an external wake-up request.

### Vercel

**Production frontend:**  
https://ine-store-price-tracker.vercel.app

Verified:

- The React/Vite frontend is deployed from the current repository.
- `VITE_API_BASE_URL` points to the production Render backend.
- The frontend loads tracked products from Supabase through the Express API.
- Search, product details, price history, scrape logs, and manual scrape flows work in the hosted application.
- Production usage confirms the Render/Vercel CORS configuration.

### GitHub Actions

**Repository:**  
https://github.com/PoonamxGit/ine-store-price-tracker

The CI workflow runs:

```text
npm ci
npm run lint
npm test
npm run build
npx playwright install --with-deps chromium
npm run test:browser
```

The latest checked workflow completed successfully after the cross-platform browser fixture encoding fix.

## External Cron

The scheduled scraper is configured through **cron-job.org**.

### Main Scrape Job

```text
Method: POST
Endpoint: https://ine-store-price-tracker-api.onrender.com/api/cron/scrape
Authentication: Authorization: Bearer <CRON_SECRET>
Schedule: every two hours
```

**Manual test result:** `202 Accepted`

The endpoint returns `202` because scraping work is queued asynchronously.

### Wake-Up Job

```text
Method: GET
Endpoint: https://ine-store-price-tracker-api.onrender.com/api/health
```

The wake-up request runs shortly before the main scrape job.

**Manual test result:** `200 OK`

The wake-up job is only a Render free-tier cold-start mitigation. It is not part of the scraping logic.

Earlier cron history contains `output too large` failures related to the previous Render service/cold-start behavior. The current endpoints were reconfigured and manual tests against the new deployment succeed.

An automatic scheduled cycle should be observed separately before claiming unattended scheduling as fully verified end-to-end.

## Real Hosted Failure and Recovery Evidence

The hosted scrape logs show real transient upstream behavior, including:

- `TIMEOUT` attempts near the configured 45-second deadline
- retries after timeout
- an upstream `429 TRY AGAIN`
- later successful attempts for the same products

This demonstrates the intended reliability behavior:

- transient failures are logged honestly
- retries are bounded
- invalid or incomplete results are not saved as valid history
- the previous successful observation remains available
- a later successful scrape can recover normally

## Headed Demonstration

The local headed command uses the same Supabase-backed scraping pipeline as the application:

```bash
npm run scrape:headed -- --product=763 --fault=fail-first
```

Observed behavior:

1. Attempt 1 intentionally aborts navigation.
2. Attempt 1 is stored as `RETRIED` with a network error.
3. Attempt 2 completes the real INE browser interaction.
4. The successful observation is validated and persisted.
5. The dashboard shows the retry/success attempt log and valid history.

The injected failure is development-only and is not enabled in production.

## Final Reproduction Checklist

A second engineer can reproduce and verify the project by following these steps:

- [ ] Clone the repository and install dependencies
- [ ] Apply `supabase/schema.sql` to a fresh Supabase project
- [ ] Configure the documented backend/frontend environment variables
- [ ] Start the local frontend and backend
- [ ] Confirm `/api/health`
- [ ] Search for an INE product
- [ ] Track a product and confirm a valid observation
- [ ] Run a manual scrape and inspect history/logs
- [ ] Run the headed `fail-first` demonstration
- [ ] Deploy the Docker backend to Render
- [ ] Deploy the Vite frontend to Vercel
- [ ] Configure the authenticated two-hour cron job
- [ ] Confirm the cron test returns `202 Accepted`

## Notes

- The deployed application does not auto-seed fake products or fallback prices.
- Test-only fixtures remain under `server/test/` and are not production data.
- Render free instances may sleep and cold-start.
- The INE demo store can occasionally respond slowly or rate-limit requests.
- These upstream failures are treated as transient failures rather than valid observations.
- Only successful validated observations are written to `price_history`.
