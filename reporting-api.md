---
layout: page
---

# Hola AI Ads — Reporting API Guide

**[Reporting API Guide](/public-docs/reporting-api/)** · [Integration Guide](/public-docs/publisher-integration/) · [Mobile SDK Guide](/public-docs/mobile-sdk-integration/) · [Payment Details Guide](/public-docs/payment-details/) · [Support](#contact)

---

This guide is for developers who want to pull their own revenue data
programmatically instead of (or in addition to) logging into the Hola AI
dashboard. Every account gets one API key, scoped to exactly the
publisher or agency it was issued for — you will only ever see your own
data.

## Authentication

Every request needs an `Authorization` header:

```
Authorization: Api-Key <your API key>
```

Your Hola AI account manager provides the key. There is no separate login
step and no token expiry — the key is valid until it's regenerated (see
below).

## Base URL

```
https://api.admin.sayhola.ai/api/public/v1/
```

## Endpoints

All endpoints are `GET`, JSON, and accept the same date-range query
parameters:

| Parameter | Values | Default |
|---|---|---|
| `range` | `24h`, `7d`, `30d`, `90d`, `1y` | `30d` |
| `start_date` / `end_date` | `YYYY-MM-DD`, used instead of `range` | — |

### `GET /reporting/summary/`

Totals for the period, plus the equivalent immediately-preceding period for
comparison.

```json
{
  "range": {"start": "2026-07-01", "end": "2026-07-29"},
  "current": {"impressions": 22250, "clicks": 79, "net_revenue": 14.608, "ctr": 0.00355, "ecpm": 0.657},
  "previous": {"impressions": 0, "clicks": 0, "net_revenue": 0.0, "ctr": 0, "ecpm": 0}
}
```

### `GET /reporting/timeseries/`

Same totals, broken out by day.

```json
{
  "range": {"start": "2026-07-01", "end": "2026-07-29"},
  "series": [
    {"date": "2026-07-01", "impressions": 780, "clicks": 3, "net_revenue": 0.52}
  ]
}
```

### `GET /reporting/breakdown/`

Totals broken out by placement.

```json
{
  "range": {"start": "2026-07-01", "end": "2026-07-29"},
  "by_placement": [
    {"placement_id": 1, "placement_name": "Banner 300x250", "impressions": 17120, "clicks": 47, "net_revenue": 9.6}
  ]
}
```

Revenue figures are always **net** (after take rate) — the same numbers
you see in the dashboard UI.

## Rate limits

| Window | Limit |
|---|---|
| Per minute | 120 requests |
| Per day | 5,000 requests |

Exceeding either returns `HTTP 429`. These limits are generous for
scheduled polling (e.g. hourly or daily pulls) — if your use case needs
more, ask your account manager.

## Code samples

### curl

```bash
curl -s "https://api.admin.sayhola.ai/api/public/v1/reporting/summary/?range=30d" \
  -H "Authorization: Api-Key tpc_key_your_key_here"
```

### Python

```python
import requests

API_KEY = "tpc_key_your_key_here"
BASE_URL = "https://api.admin.sayhola.ai/api/public/v1"

resp = requests.get(
    f"{BASE_URL}/reporting/summary/",
    params={"range": "30d"},
    headers={"Authorization": f"Api-Key {API_KEY}"},
)
resp.raise_for_status()
print(resp.json())
```

### JavaScript

```javascript
const API_KEY = "tpc_key_your_key_here";
const BASE_URL = "https://api.admin.sayhola.ai/api/public/v1";

const resp = await fetch(`${BASE_URL}/reporting/summary/?range=30d`, {
  headers: { Authorization: `Api-Key ${API_KEY}` },
});
const data = await resp.json();
console.log(data);
```

## Key rotation

If your key is ever compromised, ask your account manager to regenerate
it. Regeneration takes effect immediately — the old key stops working the
moment the new one is issued, so update your integration with the new key
before requesting a rotation.

## Contact

For a new API key, key rotation, or reporting questions, contact your
Hola AI account manager.
