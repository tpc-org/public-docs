---
layout: page
---

# Hola AI DSP — Buyer-side Integration Guide

**[Buyer-side Integration Guide](/public-docs/buyer-side-integration/)** · [Support](#getting-started)

---

This guide is for SSPs and exchanges that want to call Hola AI as a
demand source — Hola AI as the *buyer*. You send us a native bid
request, we run a real auction against our own connected demand, and
we return a net bid in your format. This is a separate integration
path from Hola AI's publisher-facing guides (server-side, web, mobile
SDK) — those are for publishers running Hola AI's own ad tech; this
one is for an SSP reselling into Hola AI's DSP.

## Endpoint & authentication

| | |
| --- | --- |
| Endpoint | `POST https://dsp.tpcsrv.com/openrtb/bid` |
| Protocol | OpenRTB 2.6 |
| Auth | `Authorization: Bearer <your token>` |
| Content-Type | `application/json` (required) |
| Optional header | `x-openrtb-version: 2.6` — if sent, must be exactly `2.6` |
| Max request size | 1 MB |

Your token is issued by Hola AI out of band — never in a request body or
URL. An alternative auth style (a shared identifier baked into the URL
as a query parameter, no `Authorization` header) is also available on
request if your platform doesn't support bearer tokens.

A request with a missing or invalid token gets `401 Unauthorized`.

## Request format

Standard OpenRTB 2.6, with two constraints:

- **Exactly one `imp` per request.** A request with zero or more than
  one `imp` is rejected (`400`).
- **That `imp` must be native** (`imp.native` present, with
  `imp.native.request` set to your Native 1.2 request as a JSON
  string). Banner and video are not currently supported — a non-native
  imp gets no bid.

Both `site` and `app` inventory are supported. The top-level `id`
field is required.

```bash
curl -X POST https://dsp.tpcsrv.com/openrtb/bid \
  -H "Authorization: Bearer <your token>" \
  -H "Content-Type: application/json" \
  -H "x-openrtb-version: 2.6" \
  -d '{
    "id": "req-12345",
    "imp": [{
      "id": "1",
      "bidfloor": 1.00,
      "bidfloorcur": "USD",
      "native": {
        "ver": "1.2",
        "request": "{\"ver\":\"1.2\",\"assets\":[{\"id\":0,\"required\":1,\"title\":{\"len\":90}},{\"id\":1,\"required\":1,\"data\":{\"type\":2,\"len\":140}},{\"id\":2,\"required\":0,\"img\":{\"type\":3,\"wmin\":300,\"hmin\":300}}]}"
      }
    }],
    "site": {"page": "https://example.com"},
    "device": {"ua": "...", "ip": "..."},
    "at": 1,
    "cur": ["USD"]
  }'
```

`bidfloor` is honored — a returned bid always clears it.

## Response format

| Status | Meaning |
| --- | --- |
| `200` | A bid. Standard OpenRTB `BidResponse` body, `cur: "USD"`. |
| `204` | No bid — empty body. The expected, lowest-overhead "no fill" response. |
| `400` | Malformed request (bad JSON, wrong imp count, non-native imp). |
| `401` | Missing or invalid auth. |

A `204` is deliberately used for every kind of no-bid — no demand
cleared your floor, the native request couldn't be matched to
available creative, a compliance gate blocked the request (see below),
or the request was shed under load. These all look identical from your
side by design: a `204` should always be treated as a normal, expected
outcome, never retried as if it were an error.

On a `200`, `seatbid[].seat` names the demand source that won
(informational — not guaranteed stable across requests) and
`bid[].adomain` is always populated.

## Native asset contract

Define your own Native 1.2 asset list in `imp.native.request` exactly
as you would for any other native demand source — your own asset `id`
numbering, your own `required` flags. The creative we return in `adm`
will use **the same asset ids and types you declared**, not a fixed
scheme of ours.

Supported asset types: `title` (text), `img` (icon or main image), and
`data` assets for description, sponsored-by/brand, and call-to-action
text.

**An asset you mark `required` must correspond to something our
connected demand can actually supply, or that request won't get a
bid.** We recommend:

- Mark `title` and one description-type `data` asset as required —
  nearly all connected demand can fill these.
- Treat `img` as optional. Not all connected demand carries creative
  imagery for every request; a required image asset with no available
  source image is the single most common reason a request that should
  otherwise fill instead gets a `204`.

If a specific asset combination consistently gets no bids where you'd
expect fills, that's the first thing worth checking with us.

## Win / loss / billing notifications

Every bid carries `nurl`, `lurl`, and `burl`, all pointing back to our
own service with standard OpenRTB macros: `${AUCTION_PRICE}` and
`${AUCTION_LOSS}`. Firing these at the right lifecycle moment is how we
learn the outcome of an auction we won — **we don't poll or ask
separately, so a bid we never hear back on is effectively invisible to
our own reporting.**

| URL | Fire when |
| --- | --- |
| `nurl` | The creative is committed for render (not the same as a viewable impression). |
| `lurl` | The bid lost — include the loss reason via `${AUCTION_LOSS}`. |
| `burl` | The impression is confirmed viewable / billable. This is our spend-recognition trigger. |

If your platform doesn't support one of these (most commonly `lurl`),
let us know — integration still works, but we lose visibility into why
a given bid didn't win.

## Compliance

A request flagged `regs.gdpr=1` with no consent string present gets
**no bid**, by default — this is a deliberate, conservative compliance
gate, not a bug. We check both the spec-correct location
(`user.consent`) and the alternate location some SSPs use in practice
(`user.ext.consent`); a TCF consent string in either field is accepted.

If your platform is GDPR/UK-GDPR/EEA-scoped and you don't currently
forward a consent string on `regs.gdpr=1` traffic, that traffic will
not receive bids from us until you do.

## Performance & rate limiting

**Timing:** budget your own request timeout generously — expect a bid
or an explicit `204` well within one second under normal conditions.
We do not currently read or honor a `tmax` value on your request, so
don't rely on us racing your specific deadline; set your own timeout
independently.

**Rate limiting:** requests past our configured throughput are shed as
a `204`, identical to a genuine no-bid, rather than an error or a
`429`. This is expected behavior under burst load. If you're seeing a
sustained, unexpected drop in fill rate that correlates with your own
traffic spikes, ask us — the limit is configurable on our side.

## Getting started

1. Contact your Hola AI account manager to request a credential and
   confirm your native asset requirements.
2. Review this guide with your integrating engineer.
3. Run a real hand-built test request against the production endpoint
   — there is currently no separate staging environment, so coordinate
   a live test window with Hola AI directly.
4. Confirm a full round trip: a `200` bid, a fired `nurl`, and (once
   the impression is viewable) a fired `burl`.
5. Ramp traffic gradually and watch fill rate before committing full
   volume.

**Support:** Contact your Hola AI account manager.
