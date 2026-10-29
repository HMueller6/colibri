# Rate Watch

A phone-and-browser dashboard for Bank of Canada rates, US Federal Reserve and bank data, bond yields (and who holds the bonds), inflation, and a rules-based read on the Canadian housing market, including a backtest of what followed past rate hikes. Everything comes from free public data. **No API key is required.**

Plain HTML, CSS and JavaScript. No build step, no dependencies. Installable as a web app (PWA) with offline support.

## Run it

```bash
cd rate-watch
node scripts/refresh.mjs     # downloads the data into data/dashboard.json (about 15 s, Node 18+)
node scripts/serve.mjs       # http://localhost:8080 (also printed: your LAN address for your phone)
```

`node scripts/refresh.mjs --check` fetches and reports per-series status without writing anything.
`node scripts/refresh.mjs --discover [group]` lists Bank of Canada Valet groups/series, which is how you repair an id if one ever moves.

## Data sources and keys

| Source | What it supplies | Key? |
|---|---|---|
| [Bank of Canada Valet API](https://www.bankofcanada.ca/valet/docs) | Policy rate, prime, benchmark GoC yields (2/3/5/7/10/long), CORRA, CPI-trim/median/common, posted mortgage rate | None |
| [Statistics Canada WDS](https://www.statcan.gc.ca/en/developers/wds) | CPI index, unemployment, new housing price index, housing starts | None |
| [FRED](https://fred.stlouisfed.org) | Fed funds target and effective rate, SOFR, Treasury yields, US CPI/core/PCE, jobs, 30-yr mortgage, breakevens, credit spreads, WTI, USD/CAD, Fed balance sheet, bank credit and mortgage delinquency, who holds US federal debt | Optional. Works without one through FRED's public CSV download. A free key ([get one](https://fredaccount.stlouisfed.org/apikeys)) in `FRED_API_KEY` switches to the official API |

Not available free: CREA/MLS resale prices and Teranet. The housing backtest therefore uses StatCan's **new** housing price index, which excludes resale homes and moves slowly. Treat its price results as a rough guide.

Hand-maintained files in `data/`: `announcement.json` (the Oct 28 decision and next date), `calendar.json`, `players.json` (a reference roster of bond-market participants, roles only, no figures). Edit `announcement.json` after each Bank of Canada decision; the dashboard cross-checks its rate against the data feed and shows a warning if they disagree.

## Putting it on your phone

An installable web app needs HTTPS. Easiest route: the workflow in `.github/workflows/rate-watch.yml` refreshes the data every 6 hours on weekdays and publishes the site to a `rate-watch-site` branch. Point Netlify, Cloudflare Pages, Vercel, or GitHub Pages ("deploy from a branch") at that branch. It must be merged to the default branch before the schedule runs; add `FRED_API_KEY` as a repository secret only if you want the official FRED API.

Then on iPhone: Share → Add to Home Screen. On Android/Chrome: menu → Install app. Offline, it shows the last data it loaded.

On the same Wi-Fi you can also open `http://<computer-ip>:8080` while `serve.mjs` runs; the app works there but cannot be installed.

## How the numbers are protected

- Every Bank of Canada series must return a label matching what we asked for; every StatCan series is title-checked when StatCan returns a title; every series' latest value must fall in a plausible range. A failure shows up on the **Data** tab and is never silently charted.
- If a series fails to refresh, the previous copy is kept and marked stale. If fewer than 70% (80% in CI) of series load, nothing is written or published.
- The 5-yr fixed rate shown is an **estimate** (GoC 5-yr yield + an editable lender spread), not a quoted rate.
- The housing read is a transparent set of rules (thresholds are shown next to each signal). It is not a forecast and not financial advice.

## Tests

```bash
node --test tests/unit.test.mjs                          # analytics, parsers and fetchers with mocked responses
node tests/make-fixture.mjs && node tests/ui.mjs [dir]   # headless-Chromium run on SYNTHETIC data (phone + desktop, light + dark, offline); needs Playwright
```

The unit tests check the mortgage maths against independently computed payments, the hike-cycle backtest against a series with a known answer, and the parsers against sample payloads. **Those payloads reflect the documented API formats but were not captured from live responses**: this project was built in a sandbox that could not reach the data hosts. Run `node scripts/refresh.mjs --check` once on a normal connection; any series whose id or format differs from expectations is reported there by name.
