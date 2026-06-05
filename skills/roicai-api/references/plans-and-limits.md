# Plans, Rate Limits & History Depth

Source of truth for how ROIC.ai meters the REST API. Use this to make any generated client
plan-aware: throttle to the requests/minute limit, keep requested date ranges inside the
history-depth limit, and handle `429` / `403` correctly.

## The model

Every plan — including Free — has **full access to all companies and every endpoint**. There is
no per-endpoint or per-ticker gating in the current model. Plans differ on exactly two axes:

1. **Rate limit** — requests per minute.
2. **History depth (lookback)** — how many years into the past you may request data for.

| Plan         | Requests / minute | History depth   | Notes |
| ------------ | ----------------- | --------------- | ----- |
| **Free**     | 5                 | 2 years         | No credit card. |
| **Individual** | 300             | 5 years         | |
| **Professional** | Unlimited\*   | 40+ yrs (all)   | Most popular. |
| **Enterprise** | Unlimited\*     | All available   | Commercial-use, multi-user. |

\* "Unlimited" means there is no published per-minute cap, but requests are still enforced
server-side against a high safety cap. Do not assume you can fire unbounded concurrency — add
sane concurrency limits and retry on `429` regardless of plan.

> Historical note for code-readers: older docs/code may mention "AAPL only on free tier" or
> `freeTier: 'Premium only'` flags. That AAPL-gating model has been **replaced** by the per-plan
> rate + lookback model above. Treat full access as the truth; the only plan differences are
> rate and history depth.

## Determining the plan

- If the user tells you their plan, size everything to it.
- If unknown, default to **Free-tier-safe**: ≤5 requests/minute and ≤2 years of history. This
  runs correctly on every plan, so it is the safe default for shared/example code.
- A live usage meter (current plan, remaining rate-limit budget, history depth) is shown to
  signed-in users at `https://www.roic.ai/api`.

## Rate-limit enforcement & headers

Exceeding the per-minute limit returns **`429 Too Many Requests`**. The response carries:

- `Retry-After` — seconds until the window resets. Use this for back-off.
- `X-RateLimit-Limit` — your plan's per-minute ceiling.
- `X-RateLimit-Remaining` — requests left in the current window.
- `X-RateLimit-Reset` — when the window resets.

Example `429` body:

```json
{
  "error": "Rate limit exceeded: max 5 requests/minute on the free plan. Retry in 42s or upgrade at https://roic.ai/pricing"
}
```

Read `X-RateLimit-Remaining` proactively and slow down before you hit zero, rather than only
reacting to `429`s.

## History-depth enforcement

Requesting data older than the plan's history depth returns **`403 Forbidden`** naming the
minimum plan that covers the range:

```json
{
  "error": "Requested history needs the Professional plan (you're on Free). Upgrade at https://roic.ai/pricing"
}
```

Avoid it by clamping the requested window to the plan:

- On financial endpoints, set `fiscal_year_start` / `date_start` no earlier than
  `currentYear − lookbackYears`.
- On price/news/splits endpoints, set `date_start` no earlier than the same boundary.

## Throttling strategy by plan

- **Free (5/min):** serialize requests; space them ~12s apart, or batch ≤5 then sleep to the
  next minute. No parallelism.
- **Individual (300/min):** token-bucket at 300/min (~5/sec). Small concurrency (e.g. 4–8) is
  fine; keep a budget from `X-RateLimit-Remaining`.
- **Professional / Enterprise (unlimited\*):** still bound concurrency (e.g. ≤20 in flight) and
  retry on `429`. Treat the cap as high-but-present.

## Plan-aware client template (Python)

Drop-in pattern that throttles to the plan, retries on `429` using `Retry-After`, and clamps the
history window. Adapt the language as needed.

```python
import os, time, datetime, requests

BASE = "https://api.roic.ai"
API_KEY = os.environ["ROIC_API_KEY"]

# Set these from the user's plan. Defaults are Free-tier-safe.
REQUESTS_PER_MINUTE = 5          # 5 Free / 300 Individual / high for Pro+Enterprise
LOOKBACK_YEARS = 2               # 2 Free / 5 Individual / None (all) for Pro+Enterprise

_min_interval = 60.0 / REQUESTS_PER_MINUTE
_last_call = 0.0

def _throttle():
    global _last_call
    wait = _min_interval - (time.monotonic() - _last_call)
    if wait > 0:
        time.sleep(wait)
    _last_call = time.monotonic()

def earliest_allowed_date():
    if LOOKBACK_YEARS is None:
        return None
    year = datetime.date.today().year - LOOKBACK_YEARS
    return f"{year}-01-01"

def get(path, params=None, max_retries=4):
    params = {**(params or {}), "apikey": API_KEY}
    # Clamp history window to the plan if the caller did not set a tighter one.
    floor = earliest_allowed_date()
    if floor and "date_start" not in params:
        params["date_start"] = floor
    for attempt in range(max_retries):
        _throttle()
        r = requests.get(f"{BASE}{path}", params=params, timeout=30)
        if r.status_code == 429:
            retry_after = int(r.headers.get("Retry-After", "5"))
            time.sleep(retry_after)
            continue
        if r.status_code == 403:
            raise RuntimeError(f"History outside plan depth: {r.json().get('error')}")
        r.raise_for_status()
        return r.json()
    raise RuntimeError("Rate limit retries exhausted")

# Example
prices = get("/v2/stock-prices/AAPL", {"limit": 30, "order": "DESC"})
income = get("/v2/fundamental/income-statement/AAPL", {"period": "annual", "limit": 5})
```

## Pricing

Plan details and upgrades: `https://www.roic.ai/pricing`.
