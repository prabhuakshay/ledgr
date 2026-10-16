# Price sources: AMFI, NSE and alternatives

Research for [#21](https://github.com/prabhuakshay/ledgr/issues/21). Checked on 2026-10-16 from a residential Indian connection with light `curl` requests, against Velora's `apps/investments/price_source.py` (URLs and parsers), `prices.py` (31-day backfill windows) and `tasks.py` (nightly fetch at 23:00 local).

## Short answer

- **The URLs are still right, but both AMFI formats have changed and Velora's parsers now read zero rows from them, without raising an error.** AMFI added `Plan` and `Option` columns to both files. AMFI's own download page says "The NAV download in the old format has been discontinued" ([AMFI NAV download](https://www.amfiindia.com/net-asset-value/nav-download)).
- NSE's bhavcopy and ETF list are live, and their formats still match what Velora parses.
- NSE is the main blocking risk. It drops requests that send a non-browser User-Agent, and requests from cloud or datacenter IPs are reported to be blocked.
- For backfill: use **AMFI's history report** for mutual funds, chunked to at most 90 days and filtered by AMC, with **mfapi.in** as the per-scheme fallback. For stocks and ETFs, use **NSE's bhavcopy archives**, with **BSE's bhavcopy** as a fallback that has the same column layout.

## Per source

### AMFI latest NAVs (`NAVAll.txt`)

- **URL:** `https://www.amfiindia.com/spages/NAVAll.txt`. It now returns a 302 redirect to `https://portal.amfiindia.com/spages/NAVAll.txt`; urllib follows the redirect automatically.
- **Status today:** 200, 1.5 MB, about 1 s. Semicolon-separated, CRLF line endings, UTF-8.
- **Format (changed):** `Scheme Code;ISIN Div Payout/ ISIN Growth;ISIN Div Reinvestment;Scheme Name;Plan;Option;Net Asset Value;Date`, which is **8 fields**. Velora's `parse_amfi` only keeps lines with exactly 6 fields, so it returns 0 of the 14,378 data lines. Velora's test fixture still uses the old 6-field header.
- **Content:** 8,717 lines carry the latest NAV date (15-Oct-2026). The rest are stale lines for wound-up or matured schemes, some dated 2018. 167 lines have no ISIN.
- **Limits and blocking:** None seen. robots.txt only disallows `/admin/`, `/login/` and `/search/`.

### AMFI NAV history report

- **URL:** `https://portal.amfiindia.com/DownloadNAVHistoryReport_Po.aspx?frmdt=DD-Mon-YYYY&todt=DD-Mon-YYYY`. It also takes an optional `mf=<AMC id>` to filter by fund house; `mf=53` is Axis.
- **Status today:** 200, `text/plain`.
- **Format (changed):** `Scheme Code;NAV Name;Plan;Option;ISIN Div Payout/ISIN Growth;ISIN Div Reinvestment;Net Asset Value;Date`. Still 8 fields, but the Repurchase and Sale price columns are gone and `Plan` and `Option` sit where Velora expects ISINs. `parse_amfi_history` therefore treats Plan and Option as ISINs, rejects them, and returns 0 rows.
- **Range limit:** AMFI's page says "maximum period of 90 days at a time". A 92-day full-market request still worked, but returned 76 MB in 74 s, which is longer than Velora's 60 s `TIMEOUT_SECONDS`. One AMC over 6 months returned 6.5 MB in 3.9 s. One full-market day is about 1.3 MB.
- **History depth:** Back to April 2006 (03-Apr-2006 returned 1,791 rows). Rows from 2006 have no ISIN. Before April 2006 the server returns **an HTML page with status 200**, not a 404.
- **Weekends:** Some schemes (overnight and liquid funds, for example) publish NAVs for weekends and holidays. Velora only asks for weekdays, which is fine.

### NSE equity bhavcopy (UDiFF)

- **URL:** `https://nsearchives.nseindia.com/content/cm/BhavCopy_NSE_CM_0_0_0_YYYYMMDD_F_0000.csv.zip`
- **Status today:** 200 for 15-Oct-2026: a 200 KB zip containing one CSV with 3,673 rows. Holidays and days not yet published return 404 (02-Oct-2026 and 16-Oct-2026 both did), as Velora expects.
- **Format (matches):** `TradDt,BizDt,Sgmt,Src,FinInstrmTp,FinInstrmId,ISIN,TckrSymb,SctySrs,…,FinInstrmNm,…,ClsPric,…`. Every column Velora reads (`ISIN`, `ClsPric`, `SctySrs`, `TradDt`, `FinInstrmNm`, `TckrSymb`) is present. Sovereign Gold Bonds come through as series `GB`.
- **History depth:** This file name works back to **01-Jan-2024**; 29-Dec-2023 and earlier return 404. Earlier days are only in the legacy archive, `content/historical/EQUITIES/YYYY/MON/cmDDMONYYYYbhav.csv.zip`, which runs up to 05-Jul-2024 (08-Jul-2024 returns 404). The legacy format is `SYMBOL,SERIES,…,CLOSE,…,TIMESTAMP,TOTALTRADES,ISIN`. The ISIN column is there in 2016 but missing in 2010.
- **Blocking:**
  - curl's default User-Agent gets its HTTP/2 stream reset. Velora's browser-like User-Agent works, and so does sending no User-Agent at all.
  - Requests from a cloud network timed out: WebFetch, which runs on a cloud network, failed on every `nsearchives` URL that home `curl` fetched instantly.
  - Users have reported NSE blocking AWS, Google Cloud and Colab IPs since August 2020 ([unofficed forum](https://forum.unofficed.com/t/nsepython-not-working-in-aws-google-cloud-and-webservers/670), [Kite forum](https://kite.trade/forum/discussion/14939/fetch-stock-in-f-o-ban-list)). Those reports mostly concern `www.nseindia.com`, and NSE has not confirmed them.
  - `www.nseindia.com/robots.txt` allows everything except `/market-data-test`.
- **Upcoming changes:** NSE circulars [NSE/MSD/74764](https://nsearchives.nseindia.com/content/circulars/MSD74764.pdf) (Jun 2026) and [NSE/MSD/76877](https://nsearchives.nseindia.com/content/circulars/MSD76877.pdf) (Oct 2026) stop the extranet and NNF-broadcast `.md`/`.ms` bhavcopies on 07-Dec-2026. They say "All other existing modes and formats of Bhavcopy dissemination will remain unchanged", so the public UDiFF CSV is not affected.

### NSE ETF list

- **URL:** `https://nsearchives.nseindia.com/content/equities/eq_etfseclist.csv`
- **Status today:** 200, 38 KB, 355 ETFs. Its header still has `ISINNumber`, the column Velora reads. Blocking risk is the same as for the bhavcopy.
- **Note:** The bhavcopy itself contains 352 rows with `INF…` ISINs, all in series `EQ` and all in the ETF list, so this extra request could be replaced by checking for an `INF` ISIN prefix.

## Alternatives for backfill

### mfapi.in

- **URL:** `https://api.mfapi.in/mf/{scheme_code}`, which optionally takes `startDate` and `endDate`. There is also `/mf/{code}/latest`, `/mf/latest`, `/mf/search?q=` and `/mf` ([OpenAPI spec](https://www.mfapi.in/docs/openapi.json)).
- **Status today:** 200, 134 KB of JSON, 0.4 s. `meta` holds the fund house, category, `isin_growth` and `isin_div_reinvestment`. `data` is a list of `{date: "DD-MM-YYYY", nav: "137.73000"}`, newest first.
- **History depth:** The full life of the scheme. Axis Midcap Direct Growth goes back to 02-Jan-2013 (its launch) with 3,402 points.
- **Limits:** The site says "No authentication, no API keys, no rate limiting" and that data updates 6× daily ([mfapi.in](https://www.mfapi.in/)).
- **Risks:** It is a one-person community project (Yuvaraj L) that mirrors AMFI. Its Terms of Service link leads nowhere (`/terms` returns 404), and there is no SLA. Requests are keyed by AMFI scheme code, not ISIN, but `NAVAll.txt` gives the mapping from ISIN to code.

### BSE equity bhavcopy (UDiFF)

- **URL:** `https://www.bseindia.com/download/BhavCopy/Equity/BhavCopy_BSE_CM_0_0_0_YYYYMMDD_F_0000.CSV` (a plain CSV, not zipped).
- **Status today:** 200, 860 KB. **It has the same UDiFF columns as NSE**, so one parser can read both.
- **Differences from NSE:**
  - Series codes are BSE's groups (`A`, `B`, `T`, `X`, `M`, …), not `EQ`/`BE`.
  - It contains 280 ETF (`INF`) rows.
- **History depth:** Back to 01-Jan-2024.
- **Gotcha:** A missing day (a holiday, or before 2024) returns **200 with a 14 KB HTML page**, not a 404. Callers have to sniff the content to tell.
- **Blocking:** It worked with no User-Agent and no Referer from a home IP. Cloud access was not tested.

## Recommendation

1. **Keep AMFI as the primary mutual fund source, with a parser that reads the header row.** Map columns by header name instead of counting fields; the header line is the first line of both files. Treat a fetch that parses to zero rows as a failure. Velora's silent zero-row result is what let this break go unnoticed.
2. **Mutual fund backfill:**
   - Call the AMFI history report with windows of at most 90 days. Keep Velora's 31-day windows, or use per-AMC requests (`mf=`) for the user's fund houses, which are much smaller.
   - Raise the timeout for full-market requests.
   - Treat any response that is not text/semicolon data (such as the pre-2006 HTML page) as "no file".
   - Use mfapi.in as a fallback, or as the faster source for a newly added scheme's full history. It is one request per scheme, but has no terms or SLA, so ledgr should not depend on it alone.
3. **Stocks and ETFs:**
   - Keep NSE's UDiFF bhavcopy.
   - Add BSE's UDiFF bhavcopy as a fallback (same parser, with content sniffing for its fake 200s).
   - Read the legacy NSE archive only if backfill before 2024 is needed.
   - Drop the ETF list request and classify ETFs by their `INF` ISIN prefix.
4. **Deployment:** Run ledgr's fetch from a residential or Indian ISP IP if possible. If ledgr is hosted on a cloud VM, expect NSE to time out, and plan to fall back to BSE or to proxying through a home connection.
