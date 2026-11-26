# PULSEGRID reliability patch

This patch is limited to the reliability, ingestion, scan scheduling, deployment packaging, and mobile-scroll items from the previously agreed repair scope.

## Changes

- REST quote, spot-candle, and futures-candle ingestion now runs concurrently in bounded pools. The Upstox client's global request gate remains the concurrency ceiling.
- Repeated WebSocket authorisation is no longer performed on every REST scan. The connection endpoint still authorises the feed when a token is connected.
- Candle recovery continues through the eligible historical-session candidates after an individual candidate fails, and records structured failures for failed attempts.
- Option-chain results are not used to select ATM strikes if no valid spot reference exists; empty chains and worker failures are explicitly recorded.
- Scan completeness now includes spot-candle failures. A failed refresh no longer resets the timestamp of a stale cached scan, and the response marks old cached data as stale.
- Server scan in-flight sharing is keyed by the active token/session cache key, preventing a request started under an older connection from being joined by a new connection or overwriting its cache.
- Browser refreshes use a recursive timeout, one active request at a time, abort-on-unmount, HTTP status checking, and request sequencing to prevent stale responses from replacing newer state.
- Mobile horizontal scrolling is restricted to actual table wrappers. Ordinary field content and the main page remain part of the document's vertical scroll flow.
- Docker packaging now uses `package*.json` and includes a `public/` directory even when the project has no public assets yet.

## Verification limits

- All 59 TypeScript/TSX source files passed a TypeScript transpilation/syntax parse.
- Full `npm run typecheck`, `npm run build`, and Vitest suites could not be run in this environment because dependencies were not installed; the package install attempt timed out.
- No live Upstox token or deployed Render environment was available, so real provider throughput, token persistence across Render restarts, and production deployment health remain unverified.
- The access token intentionally remains in process memory only. The Render blueprint remains single-instance to avoid cross-instance token/session loss; a process restart still requires reconnecting the token.


## Additional data-fetch reliability patch (2026-11-26)

- Quote batches now execute through a bounded four-worker pool rather than sequentially. Missing quote keys receive one targeted retry; any remaining missing key is retained as an explicit failure, never converted to a zero quote.
- A process-wide 429 cooldown coordinates concurrent retries to reduce retry bursts against Upstox.
- An Upstox HTTP 401 clears the unusable in-memory session immediately, preventing the UI from presenting an expired token as connected.
- After each scan, the browser reconciles its connection badge with the server-side session status. If the scan worker has no token while the browser has stale connected state, the UI returns to the reconnect flow with an explicit session-mismatch message.

## Important deployment/session constraint

The access token is deliberately held in process memory only. It cannot survive a Render restart, redeploy, process crash, or routing to a different server process. After any such event the user must paste and validate a fresh/current Upstox access token again. This patch does not persist credentials, and cannot fetch provider data while no valid token is attached. The screenshot supplied for this patch explicitly showed `NO DATA · NOT CONNECTED` / `NO FEED` and the server message `No Upstox credential attached`; that state is expected to produce empty market-data tabs until the token is connected.

## Verification

- Changed `upstox.ts` and `Dashboard.tsx` passed TypeScript transpilation/syntax checks.
- Full `tsc --noEmit` is not verified: project dependencies/types are not installed in this environment, so it reports missing `next`, `react`, `drizzle-orm`, and related declarations. No live Upstox token or Render deployment was available; provider and production behavior must still be verified after deployment.
