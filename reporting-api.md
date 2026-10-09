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
comparison. The example below is illustrative; rates are rounded.

```json
{
  "range": {"start": "2026-07-01", "end": "2026-07-29"},
  "current": {"impressions": 22250, "clicks": 79, "net_revenue": 14.608, "ad_requests": 50000, "auctions": 25000, "bid_requests": 75000, "fill_rate": 0.445, "ctr": 0.00355, "ecpm": 0.657},
  "previous": {"impressions": 0, "clicks": 0, "net_revenue": 0.0, "ad_requests": 0, "auctions": 0, "bid_requests": 0, "fill_rate": 0, "ctr": 0, "ecpm": 0}
}
```

### `GET /reporting/timeseries/`

Same totals, broken out by day. This illustrative example covers one day;
rates are rounded.

```json
{
  "range": {"start": "2026-07-01", "end": "2026-07-01"},
  "series": [
    {"date": "2026-07-01", "impressions": 780, "clicks": 3, "net_revenue": 0.52, "ad_requests": 2000, "auctions": 1000, "bid_requests": 3000, "fill_rate": 0.39, "ctr": 0.003846, "ecpm": 0.667}
  ]
}
```

The series includes every day in the requested range. Days with requests
but no impressions retain their counts and have zero fill rate, CTR and
eCPM. Empty days return zeros.

### `GET /reporting/breakdown/`

Totals broken out by placement. Match each row to your site using
`stored_imp_id`.

```json
{
  "range": {"start": "2026-10-07", "end": "2026-10-07"},
  "by_placement": [
    {"placement_id": 30, "placement_name": "Native (38f6d896)", "stored_imp_id": "wonderwall-38f6d896", "impressions": 8, "clicks": 3, "net_revenue": 0.0672, "ctr": 0.375, "ecpm": 8.4},
    {"placement_id": 31, "placement_name": "Native (4c4c401c)", "stored_imp_id": "knewz-4c4c401c", "impressions": 69, "clicks": 1, "net_revenue": 0.5796, "ctr": 0.014492753623188406, "ecpm": 8.4}
  ]
}
```

Revenue figures are always **net** (after take rate) — the same numbers
you see in the dashboard UI.

| Field | Meaning |
|---|---|
| `impressions` | Impressions reported by demand partners. A returned bid alone is not a measured impression. |
| `clicks` | Clicks reported by demand partners. |
| `net_revenue` | Revenue after Hola AI's take rate, in USD. |
| `ctr` | `clicks / impressions`, a fraction; zero when there are no impressions. |
| `ecpm` | `net_revenue / impressions × 1000`, in USD; zero when there are no impressions. |

`ctr: 0.375` means **37.5%**, and `ecpm: 8.4` means **$8.40 net per
1,000 impressions**. For a date range, each placement's rates are
calculated from its totals for the whole range. To combine placements,
calculate rates from summed clicks, impressions and net revenue rather
than averaging placement rates.

The placement breakdown does not expose request counts, `queries`, or
fill rate. Contact your account manager to agree which request and
response populations you need before interpreting those metrics.

For a single UTC day, pass the same date as `start_date` and `end_date`.
Repeated pulls return cumulative daily snapshots; replace the previously
stored daily values rather than adding each snapshot. Recent partner
reports can be restated; a scheduled pull does not guarantee finality.

## Summary and daily request counts

The summary and timeseries endpoints also return these metrics:

| Field | Meaning |
|---|---|
| `ad_requests` | Incoming ad opportunities counted per placement/format across our ad server regions, excluding shadow traffic. |
| `auctions` | Incoming physical auction calls. One call can contain several placements/formats. This is not a count of auctions won. |
| `bid_requests` | Placement requests sent to demand partners, summed across partners and regions, excluding shadow traffic. |
| `fill_rate` | `impressions / ad_requests`, capped at `1`; zero when there are no ad requests. |

`fill_rate` is a fraction: `0.10` means **10%**. It measures reported
impressions against incoming ad opportunities. It does not measure the
fraction of requests that received a priced bid. Period rates use period
totals rather than an average of daily rates.

There is no `queries` field. Agree which request population you need with
your account manager before mapping your own terminology to these counts.
Request counts and partner measurements arrive through separate ingestion
pipelines; recent values may be incomplete. A zero does not guarantee
that upstream reporting is complete.

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
