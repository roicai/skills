# ROIC.ai API — Endpoint Reference

All 31 endpoints. Base URL `https://api.roic.ai`. Every call needs `apikey` (query). Every
endpoint supports `format=json` (default) or `format=excel` (TSV). `{identifier}` accepts
ticker / CIK / CUSIP / ISIN. For rate limits, history depth, and `429`/`403` handling see
`plans-and-limits.md`.

## Contents

- [Shared financial query params](#shared-financial-query-params)
- [Tickers / discovery](#tickers--discovery) — list, search, search by name/CIK/CUSIP/ISIN, by exchange
- [Reference / market data](#reference--market-data) — exchanges, sectors, industries, countries
- [Company](#company) — profile, news
- [Earnings calls](#earnings-calls) — latest, list, transcript
- [Stock prices](#stock-prices) — historical, latest
- [Stock splits](#stock-splits) — by ticker, calendar
- [Financial statements](#financial-statements) — income, balance sheet, cash flow
- [Financial ratios](#financial-ratios) — profitability, credit, liquidity, working-capital, yield
- [Valuation](#valuation) — enterprise value, multiples, per-share

## Shared financial query params

Statements, ratios, and valuation endpoints all accept these (besides `apikey` + `format`):

| Param | Type | Default | Notes |
| ----- | ---- | ------- | ----- |
| `period` | string | `annual` | `annual` or `quarterly`. |
| `limit` | number | `10` | Max periods returned. |
| `fiscal_year_start` | number | — | Filter from this fiscal year (e.g. `2020`). |
| `fiscal_year_end` | number | — | Filter to this fiscal year (e.g. `2024`). |
| `date_start` | string | — | `YYYY-MM-DD`. Use to clamp to plan history depth. |
| `date_end` | string | — | `YYYY-MM-DD`. |
| `order` | string | `DESC` | `ASC` or `DESC`. |

---

## Tickers / discovery

Resolve names/identifiers to tickers and enumerate the tradable universe. All return live data.

### List Tickers — `GET /v2/tickers/list`
Complete list of every ticker: symbol, name, exchange, type, listing status.
**Params:** `apikey`, `format`, `listed` (bool, optional — `true` listed / `false` delisted; omit for all).
**Use when:** building a local ticker DB synced daily, autocomplete source, the base universe for a
screener, or a compliance master list. Export with `format=excel`.
**Returns:** array of `{ symbol, name, exchange_name, exchange, type, listed }`.

### Search Tickers — `GET /v2/tickers/search`
Search by symbol or company name across all exchanges.
**Params:** `query` (required), `limit` (default 10), `exchange` (optional), `apikey`, `format`.
**Use when:** real-time search bar/autocomplete, validating a user-entered symbol before downstream
calls, resolving a name to a symbol.
**Returns:** array of `{ symbol, name, exchange_name, exchange, type }`.

### Search Tickers by Company Name — `GET /v2/tickers/search/name`
Name-only search.
**Params:** `query` (required), `limit` (default 10), `exchange` (optional), `apikey`, `format`.
**Use when:** company-name search box, resolving names from articles/documents to symbols.

### Search Ticker by CIK — `GET /v2/tickers/search/cik/{cik}`
Look up a ticker by SEC CIK (Central Index Key).
**Params:** `cik` (path, required, e.g. `0000320193`), `apikey`, `format`.
**Use when:** mapping SEC EDGAR filings to tickers, enriching regulatory datasets keyed by CIK.
**Returns:** `{ cik, symbol, name, exchange_name, exchange, type }`.

### Search Ticker by CUSIP — `GET /v2/tickers/search/cusip/{cusip}`
Look up a ticker by 9-char CUSIP.
**Params:** `cusip` (path, required, e.g. `037833100`), `apikey`, `format`.
**Use when:** reconciling brokerage/custodian holdings identified by CUSIP.

### Search Ticker by ISIN — `GET /v2/tickers/search/isin/{isin}`
Look up a ticker by 12-char ISIN.
**Params:** `isin` (path, required, e.g. `US0378331005`), `apikey`, `format`.
**Use when:** normalizing international portfolios that use ISINs.

### List Tickers by Exchange — `GET /v2/tickers/search/exchange/{exchange}`
All tickers listed on one exchange.
**Params:** `exchange` (path, required, e.g. `NASDAQ`), `apikey`, `format`.
**Use when:** exchange-filtered screener, exchange-specific watchlists, market-composition analysis.

---

## Reference / market data

Live lists with coverage counts. Use to populate filters and validate codes before querying.

### List All Exchanges — `GET /v2/exchanges/list`
**Params:** `apikey`, `format`. **Returns:** `{ exchange, exchange_name, ticker_count }[]`.
**Use when:** exchange filter dropdown, discovering coverage before deeper queries.

### List All Sectors — `GET /v2/sectors/list`
**Params:** `apikey`, `format`. **Returns:** `{ sector, ticker_count }[]`.
**Use when:** sector filter dropdowns, sector-allocation charts, validating sector names.

### List All Industries — `GET /v2/industries/list`
**Params:** `apikey`, `format`. **Returns:** `{ industry, ticker_count }[]`.
**Use when:** industry-level filters, peer-group discovery, diversification analysis.

### List All Countries — `GET /v2/countries/list`
**Params:** `apikey`, `format`. **Returns:** `{ country, country_code, ticker_count }[]`.
**Use when:** country filter for a global screener, geographic distribution maps.

---

## Company

### Get Company Profile — `GET /v2/company/profile/{identifier}`
Business description, sector, industry, market cap, CEO, employees, IPO date, dividend info,
website, country, currency — 30+ data points.
**Params:** `identifier` (path), `apikey`, `format`.
**Use when:** company overview pages, enriching search results with context, feeding metadata to AI
analysis tools.

### Get Company News — `GET /v2/company/news/{identifier}`
Latest news articles: title, URL, full text, date, source. Paginated + date-filterable.
**Params:** `identifier` (path), `limit` (default 20), `page` (default 0, zero-indexed),
`date_start`, `date_end`, `apikey`, `format`.
**Use when:** news feed for holdings, NLP sentiment/trading signals, monitoring around earnings,
feeding articles to AI summarizers.
**Returns:** `{ symbol, title, article_url, article_text, published_date, site }[]`.

---

## Earnings calls

### Get Latest Earnings Call — `GET /v2/company/earnings-calls/latest/{identifier}`
Most recent transcript with date, fiscal year, quarter, full `content`.
**Params:** `identifier` (path), `apikey`, `format`.
**Use when:** earnings-alert systems, AI summaries of the newest call, portfolio monitoring.

### List Earnings Calls — `GET /v2/company/earnings-calls/list/{identifier}`
Available transcripts (year, quarter, date) — no body text.
**Params:** `identifier` (path), `limit` (default 100), `apikey`, `format`.
**Use when:** transcript navigation dropdowns, coverage checks before batch jobs, setting up
multi-year fetches.
**Returns:** `{ symbol, year, quarter, date }[]`.

### Get Earnings Call Transcript — `GET /v2/company/earnings-calls/transcript/{identifier}`
A specific transcript by year + quarter.
**Params:** `identifier` (path), `year` (required), `quarter` (required, 1–4), `apikey`, `format`.
**Use when:** quarter-over-quarter commentary analysis, event studies, targeted NLP extraction,
peer comparison of the same quarter.

---

## Stock prices

### Get Historical Stock Prices — `GET /v2/stock-prices/{identifier}`
Daily OHLCV + adjusted close + VWAP, date-range filterable and sortable.
**Params:** `identifier` (path), `limit` (default 100), `date_start`, `date_end`,
`order` (default DESC), `apikey`, `format`.
**Use when:** price charts, backtests, adjusted-close return calcs, technical indicators, event
studies, volatility analysis. `format=excel` + Sheets `IMPORTDATA()` for live models.
**Returns:** array of `{ date, open, high, low, close, adj_close, volume, unadjusted_volume,
change, change_percent, vwap, label }`.

### Get Latest Stock Price — `GET /v2/stock-prices/latest/{identifier}`
Most recent trading-day data.
**Params:** `identifier` (path), `apikey`, `format`.
**Use when:** portfolio dashboards, watchlist/price-alert polling, price widgets, quick lookups,
batch daily-mover scans.

---

## Stock splits

### Get Stock Splits by Ticker — `GET /v2/stock-splits/{identifier}`
Full split history for one ticker: ratio, factor, reverse flag, pre/post share counts + prices.
Sorted by split date desc. Cache: daily.
**Params:** `identifier` (path), `apikey`, `format`.
**Use when:** back-adjusting OHLCV via `split_factor`, long-term return math, reverse-split
screening, share-count/cost-basis reconciliation, corporate-action audit.
**Returns:** array of `{ id, symbol, split_date, split_ratio:{to,from}, split_factor,
is_reverse_split, pre_split_shares, post_split_shares, pre_split_price, post_split_price, currency }`.

### Get Stock Splits Calendar — `GET /v2/stock-splits`
Upcoming + historical splits across **all** tickers in one feed. Cache: 4h.
**Params:** `limit` (default 100, max 10000), `date_start`, `date_end`, `order` (default DESC),
`apikey`, `format`.
**Use when:** event studies, pre-adjusting raw data before backtests, brokerage/PMS reconciliation,
reverse-split monitoring, corporate-actions widgets/briefings.

---

## Financial statements

All accept the [shared financial query params](#shared-financial-query-params). Returns arrays of
period objects keyed `ticker, date, period, period_label, fiscal_year, currency` plus line items.

### Get Income Statement — `GET /v2/fundamental/income-statement/{identifier}`
Revenue, COGS, gross profit, operating income, net income, EPS (basic/diluted), EBITDA/EBIT,
margins, dividends per share — 40+ line items (`is_*`, `eps`, `ebitda`, `*_margin`).
**Use when:** DCF inputs, revenue-growth tracking, profitability screens, earnings-quality checks,
peer margin benchmarking, AI earnings reports, Sheets models.

### Get Balance Sheet — `GET /v2/fundamental/balance-sheet/{identifier}`
Assets, liabilities, equity, cash, total debt, inventories — 75+ items (`bs_*`) plus `net_debt`,
`cur_ratio`, `cash_conversion_cycle`, `tce_ratio`.
**Use when:** leverage ratios, liquidity/solvency, working-capital trends, capital-structure
analysis, NAV/book-value models, financial-health screens, AI credit analysis.

### Get Cash Flow Statement — `GET /v2/fundamental/cash-flow/{identifier}`
Operating cash flow, capex, free cash flow, dividends paid, buybacks — 50+ items (`cf_*`) plus
`cf_free_cash_flow`, `free_cash_flow_per_sh`, `pr_to_free_cash_flow`.
**Use when:** DCF from historical FCF, capital-allocation analysis, earnings quality (NI vs OCF),
FCF-yield screens, capex intensity, dividend-coverage/burn-rate tracking.

---

## Financial ratios

Pre-calculated — prefer these over re-deriving from raw statements. All accept the
[shared financial query params](#shared-financial-query-params); same period-object envelope.

### Get Profitability Ratios — `GET /v2/fundamental/ratios/profitability/{identifier}`
ROE (`return_com_eqy`), ROA, ROC/ROIC, gross/EBITDA/operating/profit margins, effective tax rate,
payout ratio, sustainable growth rate.
**Use when:** quality screens (high ROE + steady margins), pricing-power comparison, margin-trend
tracking, sustainable-growth inputs.

### Get Credit & Debt Ratios — `GET /v2/fundamental/ratios/credit/{identifier}`
Debt-to-EBITDA, net-debt-to-EBITDA, debt-to-equity, debt-to-capital/asset, interest-coverage
context, EBITDA after capex.
**Use when:** conservative-balance-sheet screens, distress monitoring, peer leverage comparison,
deleveraging tracking, credit-scoring features.

### Get Liquidity Ratios — `GET /v2/fundamental/ratios/liquidity/{identifier}`
Cash ratio, current ratio, quick ratio, CFO-to-current-liabilities, and `altman_z_score`.
**Use when:** filtering out current-ratio < 1 names, detecting deteriorating short-term health,
Altman-Z bankruptcy screening, sector liquidity comparison.

### Get Working Capital Ratios — `GET /v2/fundamental/ratios/working-capital/{identifier}`
Receivables/inventory/payables turnover (+ days), cash conversion cycle, asset turnover.
**Use when:** capital-efficiency peer comparison, inventory/receivables-trend detection, DuPont
decomposition, negative-CCC operational-excellence screens.

### Get Yield Analysis Ratios — `GET /v2/fundamental/ratios/yield-analysis/{identifier}`
TTM cash-flow yield, FCF yield, shareholder yield, buyback yield, capital yield.
**Use when:** high-FCF-yield value screens, shareholder-yield (dividends + buybacks) strategies,
buyback-trend tracking, accounting-resistant valuation vs P/E.

---

## Valuation

All accept the [shared financial query params](#shared-financial-query-params).

### Get Enterprise Value Metrics — `GET /v2/fundamental/enterprise-value/{identifier}`
Market cap, enterprise value, cash, debt, EV/TTM-sales/EBITDA/EBIT/cash-flow, diluted EV, TTM
sales/EBITDA/operating-income/cash-flow.
**Use when:** EV/EBITDA + EV/Sales comparables, EV-over-time M&A research, DCF starting points,
capital-structure-neutral cross-sector comparison.

### Get Valuation Multiples — `GET /v2/fundamental/multiples/{identifier}`
P/E, P/B, P/S, P/CF, P/FCF, EV/Sales, EV/EBITDA, EV/EBIT — each with average + high/low variants,
plus last/high/low price and EV.
**Use when:** multiple-based screeners, comparable-company tables, valuation expansion/compression
tracking, over/undervalued-vs-range detection, valuation dashboards.

### Get Per-Share Data — `GET /v2/fundamental/per-share/{identifier}`
EPS (basic/diluted), book & tangible book value per share, revenue/EBITDA/operating-income/
cash-flow/FCF per share, dividends per share, shares outstanding.
**Use when:** custom ratios (per-share × price), EPS-growth tracking, dividend-per-share analysis,
per-share peer tables, DCF/DDM per-share inputs.
