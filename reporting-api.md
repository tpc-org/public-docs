---
layout: page
---

# Hola AI Ads — Reporting API Guide

**[Reporting API Guide](/public-docs/reporting-api/)** · [Integration Guide](/public-docs/publisher-integration/) · [Mobile SDK Guide](/public-docs/mobile-sdk-integration/) · [Server-side Guide](/public-docs/server-side-integration/) · [Payment Details Guide](/public-docs/payment-details/) · [Support](#contact)

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
  "current": {"impressions": 22250, "clicks": 79, "net_revenue": 14.608, "ad_requests": 50000, "auctions": 25000, "bid_requests": 75000, "fill_rate": 0.445, "ctr": 0.00355, "ecpm": 0.657},
  "previous": {"impressions": 0, "clicks": 0, "net_revenue": 0.0, "ad_requests": 0, "auctions": 0, "bid_requests": 0, "fill_rate": 0, "ctr": 0, "ecpm": 0}
}
```

### `GET /reporting/timeseries/`

Same totals, broken out by day.

```json
{
  "range": {"start": "2026-07-01", "end": "2026-07-29"},
  "series": [
    {"date": "2026-07-01", "impressions": 780, "clicks": 3, "net_revenue": 0.52, "ad_requests": 2000, "auctions": 1000, "bid_requests": 3000, "fill_rate": 0.39, "ctr": 0.003846, "ecpm": 0.667}
  ]
}
```

The series includes every day in the requested range, including days with
no records. Days with requests but no impressions retain their request
counts and have zero fill rate, CTR and eCPM.

### `GET /reporting/breakdown/`

Totals broken out by placement.

```json
{
  "range": {"start": "2026-07-01", "end": "2026-07-29"},
  "by_placement": [
    {"placement_id": 1, "placement_name": "Banner 300x250", "stored_imp_id": "publisher-banner", "impressions": 17120, "clicks": 47, "net_revenue": 9.6, "ctr": 0.002745, "ecpm": 0.561}
  ]
}
```

Revenue figures are always **net** (after take rate) — the same numbers
you see in the dashboard UI.

Match placements using `stored_imp_id`. Each placement includes `ctr`
(`clicks / impressions`, a fraction) and net `ecpm`
(`net_revenue / impressions × 1000`, in USD). Both are zero when
impressions are zero. For example, 8 impressions, 3 clicks, and $0.0672
net revenue produce `ctr: 0.375` (37.5%) and `ecpm: 8.4`.

For a single UTC day, pass the same date as `start_date` and `end_date`.
Repeated pulls return cumulative daily snapshots; replace the previously
stored daily values rather than adding each snapshot. Recent partner
reports can be restated; a scheduled pull does not guarantee finality.

## Summary and daily metric definitions

| Field | Meaning |
|---|---|
| `ad_requests` | Ad opportunities received by our ad server, counted per placement. |
| `auctions` | Auction calls; one call can contain several placements, so this differs from `ad_requests`. |
| `bid_requests` | Placement requests sent to configured demand partners, summed across partners. |
| `impressions` | Impressions reported by demand partners. A returned bid alone is not a measured impression. |
| `clicks` | Clicks reported by demand partners. |
| `net_revenue` | Revenue after Hola AI's take rate, in USD. |
| `fill_rate` | `impressions / ad_requests`, capped at `1`; zero when there are no ad requests. |
| `ctr` | `clicks / impressions`; zero when there are no impressions. |
| `ecpm` | `net_revenue / impressions × 1000`, in USD; zero when there are no impressions. |

`ctr` and `fill_rate` are ratios, not percentage values: `0.10` means
**10%**. Example values above are rounded for readability. Period rates
are calculated from period totals, not by averaging daily rates.

Use the existing summary or timeseries `ad_requests` field for request volume;
there is no `queries` field. Confirm whether your own "queries" metric
counts placements or auction calls before mapping it to this API.

These count and rate fields apply to summary and timeseries reports.
The placement breakdown includes impressions, clicks, net revenue, CTR,
and net eCPM. It does not expose request counts or fill rate.
Request counts and partner measurements come from separate
ingestion pipelines; recent days can be incomplete, and zero values do
not guarantee that upstream reporting is complete.

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
