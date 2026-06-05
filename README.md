# roicai-api skill

Agent skill for the [ROIC.ai Financial Data API](https://www.roic.ai/api). Gives an AI coding
assistant everything it needs to build, call, and debug integrations against
`https://api.roic.ai` — correct endpoint paths, parameters, response shapes, and **plan-based
rate limits** (requests/minute + history depth), so generated client code works in production
instead of failing with `429`/`403`.

Covers all 31 endpoints across 9 groups: tickers/discovery, market-data reference, company
profile & news, earnings-call transcripts, stock prices, stock splits, financial statements
(income / balance sheet / cash flow), pre-calculated ratios (profitability, credit, liquidity,
working-capital, yield), and valuation (enterprise value, multiples, per-share).

## When to use

Use this skill whenever you (or an AI assistant) are:

- Building a project that pulls financial data — stock prices, fundamentals, ratios, valuation,
  earnings transcripts, or company/market reference data — from ROIC.ai.
- Writing client code that calls `api.roic.ai` and needs the right endpoint, params, and auth.
- Picking **which** endpoint fits the data you want (e.g. "current price" → `stock-prices/latest`,
  "ROE/ROIC" → `ratios/profitability`, "P/E, EV/EBITDA" → `multiples`).
- Making a client **plan-aware**: sizing throttle/back-off to your plan's requests-per-minute
  limit and clamping date ranges to your history-depth limit.
- Debugging `401` (bad key), `403` (history outside plan), `429` (rate limit — honor
  `Retry-After` / `X-RateLimit-*` headers), or `404` (unknown identifier).
- Exporting data to Google Sheets / Excel via `format=excel` + `IMPORTDATA()`.

Trigger words: `api.roic.ai`, "roic api", "roic.ai api", "financial data api", an `apikey` query
param, or any request to fetch fundamentals/prices for a project.

## Contents

```
api/roicai-api/
├── SKILL.md                         # overview, auth, plan limits, endpoint index, errors
└── references/
    ├── endpoints.md                 # all 31 endpoints: params, when-to-use, response shape
    └── plans-and-limits.md          # rate limits, history depth, 429/403, plan-aware client
```

## Install

Copy the skill into your agent's skills directory:

```bash
cp -r api/roicai-api ~/.claude/skills/roicai-api
```

## Plans & rate limits (summary)

| Plan         | Requests / minute | History depth |
| ------------ | ----------------- | ------------- |
| Free         | 5                 | 2 years       |
| Individual   | 300               | 5 years       |
| Professional | Unlimited\*       | 40+ yrs (all) |
| Enterprise   | Unlimited\*       | All available |

\* No published per-minute cap; still enforced server-side against a high safety cap.
Full detail in [`references/plans-and-limits.md`](api/roicai-api/references/plans-and-limits.md).

## Links & SEO

Built by **[ROIC.ai](https://www.roic.ai)** — financial data API for 60,000+ public companies across
70+ global exchanges.

- 🌐 Website: <https://www.roic.ai>
- 📚 API docs: <https://www.roic.ai/api>
- 🔑 Authentication: <https://www.roic.ai/api/docs/authentication>
- 💳 Pricing & plans: <https://www.roic.ai/pricing>
- 🤖 MCP server (AI assistants): <https://www.roic.ai/api/mcp>
- ✉️ Support: support@roic.ai
