---
name: roicai-api
description: >-
  ROIC.ai Financial Data REST API integration reference. Use this skill whenever the
  user builds, calls, or debugs an integration against the ROIC.ai financial-data API
  (base URL `https://api.roic.ai`) — pulling stock prices, financial statements
  (income/balance/cash-flow), pre-calculated ratios (profitability, credit, liquidity,
  working-capital, yield), valuation multiples, enterprise value, per-share data,
  earnings-call transcripts, company profiles/news, stock splits, or ticker / exchange /
  sector / industry / country reference data. Trigger on `api.roic.ai`, an `apikey` query
  param against roic, mentions of "roic api" / "roic.ai api" / "financial data api", or any
  request to write client code that fetches fundamentals or prices and needs correct
  endpoint paths, parameters, and plan-based rate limits. ALWAYS use it — even when the user
  does not name the endpoint — to map the data they want to the right endpoint AND to size
  throttling/back-off and history windows to their plan (requests/minute limit, lookback-year
  limit, 429 + Retry-After + X-RateLimit-* headers, 403 history-depth gating).
---

# ROIC.ai Financial Data API

REST API for financial data on 60,000+ public companies across 70+ global exchanges:
stock prices, financial statements, pre-calculated ratios, valuation, earnings-call
transcripts, company profiles/news, stock splits, and reference data.

**Base URL:** `https://api.roic.ai`
**Auth:** every request needs an `apikey` query parameter.
**Format:** every endpoint returns JSON (default) or TSV for spreadsheets via `format=excel`.

When helping someone integrate this API, two things matter beyond getting the path right:
pick the **right endpoint** for the data they want (see the index below + `references/endpoints.md`),
and make the client **plan-aware** so it does not blow past the rate limit or request history
the plan does not cover (see `references/plans-and-limits.md`). Both are core to this skill.

## Authentication

Pass the key as a query param on any endpoint. Never hardcode it in client-side code or
commit it — keep it in an env var / secret manager and proxy calls server-side.

```bash
curl "https://api.roic.ai/v2/stock-prices/latest/AAPL?apikey=YOUR_API_KEY"
```

`401 Unauthorized` = missing/invalid key.

## Plans, rate limits & history depth (always account for this)

Every plan has full access to **all companies and all endpoints**. Plans differ only by
**rate limit** (requests/minute) and **history depth** (how many years back you may query).
This is the single most important thing to get right when generating client code — under-plan
code silently fails in production with `429`/`403`.

| Plan         | Requests / minute | History depth |
| ------------ | ----------------- | ------------- |
| Free         | 5                 | 2 years       |
| Individual   | 300               | 5 years       |
| Professional | Unlimited\*       | 40+ yrs (all) |
| Enterprise   | Unlimited\*       | All available |

\* "Unlimited" is enforced server-side against a high safety cap, not the absence of a limit.

**Rules of thumb when writing code:**

- **Ask or assume the plan.** If the user states a plan, size everything to it. If unknown,
  default to **Free-tier-safe** behavior (≤5 req/min, ≤2 yr lookback) — it works on every plan.
- **Throttle proactively.** On Free, serialize requests and space them ≥12s apart (or batch
  ≤5/min). On Individual, keep a token-bucket ≤300/min. On Pro/Enterprise, still add modest
  concurrency limits + retry — "unlimited" is capped.
- **Handle `429 Too Many Requests`.** Respect the `Retry-After` header (seconds) for back-off;
  read `X-RateLimit-Limit` / `X-RateLimit-Remaining` / `X-RateLimit-Reset` to self-pace.
- **Handle `403 Forbidden`.** Returned when the requested date range is older than the plan's
  history depth. Either clamp the request window to the plan, or surface the upgrade path. Use
  `date_start` / `fiscal_year_start` to stay inside the window.

Full detail, error bodies, and a plan-aware client template: `references/plans-and-limits.md`.

## Conventions shared by all endpoints

- **`apikey`** (required, query) — auth on every call.
- **`format`** (optional, query) — `json` (default) or `excel` (TSV; pairs with Google Sheets
  `=IMPORTDATA(...)` and Excel).
- **`identifier`** (path, on company/financial endpoints) — accepts ticker (`AAPL`), SEC CIK,
  CUSIP, or ISIN interchangeably.
- **Financial endpoints** (statements, ratios, valuation) share these query params:
  `period` (`annual` default / `quarterly`), `limit` (periods to return, default 10),
  `fiscal_year_start`, `fiscal_year_end`, `date_start`, `date_end` (YYYY-MM-DD),
  `order` (`DESC` default / `ASC`).
- **Time-series endpoints** (prices, news, splits) support `date_start`/`date_end`, `limit`,
  and usually `order`; news also paginates with `page` (zero-indexed).

## Endpoint index

31 endpoints in 9 groups. Match the user's need to a group, then open
`references/endpoints.md` for that endpoint's exact params, use cases, and response shape.

| Group | Endpoints (path under `https://api.roic.ai`) |
| ----- | -------------------------------------------- |
| **Tickers / discovery** | `GET /v2/tickers/list`, `/v2/tickers/search`, `/v2/tickers/search/name`, `/v2/tickers/search/cik/{cik}`, `/v2/tickers/search/cusip/{cusip}`, `/v2/tickers/search/isin/{isin}`, `/v2/tickers/search/exchange/{exchange}` |
| **Reference / market data** | `GET /v2/exchanges/list`, `/v2/sectors/list`, `/v2/industries/list`, `/v2/countries/list` |
| **Company** | `GET /v2/company/profile/{identifier}`, `/v2/company/news/{identifier}` |
| **Earnings calls** | `GET /v2/company/earnings-calls/latest/{identifier}`, `/v2/company/earnings-calls/list/{identifier}`, `/v2/company/earnings-calls/transcript/{identifier}` |
| **Stock prices** | `GET /v2/stock-prices/{identifier}`, `/v2/stock-prices/latest/{identifier}` |
| **Stock splits** | `GET /v2/stock-splits/{identifier}` (by ticker), `/v2/stock-splits` (calendar) |
| **Financial statements** | `GET /v2/fundamental/income-statement/{identifier}`, `/v2/fundamental/balance-sheet/{identifier}`, `/v2/fundamental/cash-flow/{identifier}` |
| **Financial ratios** | `GET /v2/fundamental/ratios/profitability/{identifier}`, `/credit/`, `/liquidity/`, `/working-capital/`, `/yield-analysis/` `{identifier}` |
| **Valuation** | `GET /v2/fundamental/enterprise-value/{identifier}`, `/v2/fundamental/multiples/{identifier}`, `/v2/fundamental/per-share/{identifier}` |

**Choosing the right one (common cases):**

- "current price" → `stock-prices/latest`; "price chart / backtest" → `stock-prices` (historical).
- "revenue / margins / EPS" → `income-statement`; "debt / assets / equity" → `balance-sheet`;
  "free cash flow / capex / dividends" → `cash-flow`.
- "ROE/ROA/ROIC, margins" → `ratios/profitability`; "leverage, debt-to-EBITDA, interest cover"
  → `ratios/credit`; "current/quick ratio, Altman Z" → `ratios/liquidity`; "turnover, cash
  conversion cycle" → `ratios/working-capital`; "FCF yield, shareholder/buyback yield" →
  `ratios/yield-analysis`.
- "P/E, P/B, EV/EBITDA" → `multiples`; "market cap / EV / TTM metrics" → `enterprise-value`;
  "EPS, book value, dividends per share" → `per-share`.
- "resolve a name/CIK/CUSIP/ISIN to a ticker" → the matching `tickers/search*` endpoint.
- "what exchanges/sectors/industries/countries exist" → the reference list endpoints.

Pre-calculated ratios are a differentiator — prefer the `ratios/*` and `multiples`/`per-share`
endpoints over re-deriving metrics client-side from raw statements.

## Code patterns

A plan-aware request with retry/back-off (Python) and other languages' base snippets live in
`references/plans-and-limits.md` and `references/endpoints.md`. Minimal calls:

```bash
# JSON
curl "https://api.roic.ai/v2/fundamental/income-statement/AAPL?apikey=YOUR_API_KEY&period=annual&limit=5"
# Spreadsheet (Google Sheets cell)
=IMPORTDATA("https://api.roic.ai/v2/stock-prices/latest/AAPL?format=excel&apikey=YOUR_API_KEY")
```

```javascript
const r = await fetch(
  `https://api.roic.ai/v2/stock-prices/latest/AAPL?apikey=${process.env.ROIC_API_KEY}`,
);
if (r.status === 429) {
  /* back off using Retry-After header, then retry */
}
const data = await r.json();
```

## Errors

| Status | Meaning | Action |
| ------ | ------- | ------ |
| `200` | OK | — |
| `401 Unauthorized` | Missing/invalid API key | Fix the `apikey` param. |
| `403 Forbidden` | Requested history outside plan's depth | Clamp window or upgrade. |
| `404 Not Found` | Company/resource not found | Validate identifier via `tickers/search`. |
| `429 Too Many Requests` | Rate limit exceeded | Back off per `Retry-After`; throttle to plan. |
| `500 Internal Server Error` | Server error | Retry with back-off; contact support@roic.ai. |

## When to read the reference files

- `references/endpoints.md` — every endpoint's method, path, full parameter list, when-to-use
  scenarios, and response field shape. Open it whenever you need exact params or are unsure
  which endpoint fits.
- `references/plans-and-limits.md` — exact rate-limit / history-depth semantics, error response
  bodies, rate-limit headers, and a copy-paste plan-aware client (throttle + retry). Open it
  whenever you generate client code that makes more than a one-off request.

## Alternative: MCP server (AI assistants)

For connecting an AI assistant (Claude, ChatGPT, Copilot, Codex) in natural language rather than
building a REST integration, ROIC.ai also exposes an MCP server at `https://mcp.roic.ai/mcp`
(OAuth 2.1, no API key). Same data, different surface. The REST API in this skill is the path
for custom code/integrations.
