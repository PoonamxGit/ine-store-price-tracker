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
