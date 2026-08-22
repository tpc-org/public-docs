---
layout: page
---

# Hola AI Ads — Server-side Integration Guide

**[Integration Guide](/public-docs/server-side-integration/)** · [Web Integration Guide](/public-docs/publisher-integration/) · [Mobile SDK Guide](/public-docs/mobile-sdk-integration/) · [Reporting API Guide](/public-docs/reporting-api/) · [Payment Details Guide](/public-docs/payment-details/) · [Support](#contact)

---

This guide is for publishers whose own backend calls our Prebid Server
directly — raw OpenRTB2 over HTTP, no browser, no Prebid.js bundle, no SDK.
This is a separate integration path from the web `<script>` tag guide and
the Mobile SDK guide. If your site or app already uses one of those, you
don't need this guide — but you can mix and match: it's common for a
publisher to run the web bundle on their main site, a native app via the
Mobile SDK, and their own backend-to-backend integration for a separate
surface (a newsletter renderer, an AMP page, a partner feed), all at once.

You'll need: our Prebid Server endpoint, your account ID, your auction
stored request ID, and one config ID per ad placement. Your Hola AI account
manager provides these.

## Quick start

There are two halves to this integration, because they run in two different
places:

1. **Your backend runs the auction** — sends a request to our Prebid Server,
   gets back a winning bid (or no bid).
2. **The visitor's browser renders the ad** — your backend hands the bid to
   the page however fits your stack, and the page renders the creative and
   fires its trackers.

### 1. Run the auction request

Complete, runnable examples in three languages. Each builds the same
request: an OpenRTB2 `site` object, your placement's config ID, your
account's auction stored request ID, and — for native placements — one
generic per-auction content object under `ext.prebid.bidder.tpc`
(`accountId` + `userId` + `sessionId` + `messages`, the same shape as the
web integration's `window.tpc.data`). Our server resolves that into
whatever real demand is actually configured for the placement — you never
send anything demand-source-specific yourself, and adding, removing, or
swapping demand behind a placement never requires a change on your side.
See "Contextual / chat-native ads" below.

**Node.js** (no dependencies — uses the built-in `fetch`):

```javascript
const AUCTION_URL = 'https://pbs.tpcsrv.com/openrtb2/auction';
const AUCTION_STORED_REQUEST_ID = '<your-auction-stored-request-id>';
const NATIVE_CONFIG_ID = '<your-config-id>';
const ACCOUNT_ID = '<your-account-id>';

// 0=title, 1=image, 2=description, 3=sponsor, 4=CTA
const NATIVE_REQUEST = JSON.stringify({
  ver: '1.2',
  assets: [
    { id: 0, required: 1, title: { len: 80 } },
    { id: 1, required: 0, img: { type: 3, wmin: 1, hmin: 1 } },
    { id: 2, required: 0, data: { type: 2, len: 100 } },
    { id: 3, required: 0, data: { type: 1 } },
    { id: 4, required: 0, data: { type: 12, len: 30 } },
  ],
});

// Same shape as the web integration's window.tpc.data — one object per
// conversation, sent as-is under ext.prebid.bidder.tpc. `userId` is a
// stable per-user identifier and `sessionId` a per-session one: generate
// and persist both yourself (e.g. a UUID per visitor, and a second UUID
// per conversation/session) — the web bundle auto-generates both when
// omitted, but there's no bundle here to do that for you. `messages`
// grows by one entry each turn; the assistant entry is only added once a
// reply exists. Our server resolves this into whatever demand is actually
// configured for the placement — there's no partner-specific mapping to
// do on your side.
const tpcData = {
  accountId: ACCOUNT_ID,
  userId: 'stable-per-user-id',
  sessionId: 'stable-per-session-id',
  messages: [
    { role: 'user', content: "I'm looking for new shoes" },
    // { role: 'assistant', content: '...' } — add once your reply exists
  ],
};

async function runAuction() {
  const requestBody = {
    id: `req-${Date.now()}`,
    site: { page: 'https://your-site.example.com/article' },
    imp: [
      {
        id: '1',
        native: { request: NATIVE_REQUEST, ver: '1.2' },
        ext: {
          prebid: {
            storedrequest: { id: NATIVE_CONFIG_ID },
            bidder: { tpc: tpcData },
          },
        },
      },
    ],
    ext: { prebid: { storedrequest: { id: AUCTION_STORED_REQUEST_ID } } },
    // Consent — see "Consent" below.
    regs: { ext: { gdpr: 0 } },
    user: { ext: { consent: '' } },
  };

  const res = await fetch(AUCTION_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(requestBody),
  });
  const data = await res.json();

  const seatbid = data.seatbid || [];
  if (seatbid.length === 0) {
    console.log('No bid returned.');
    return;
  }
  const bid = seatbid[0].bid[0];
  console.log('Winning bid price:', bid.price);
  console.log('adm:', bid.adm);
}

runAuction();
```

**Go** (standard library only):

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"time"
)

const (
	auctionURL             = "https://pbs.tpcsrv.com/openrtb2/auction"
	auctionStoredRequestID = "<your-auction-stored-request-id>"
	nativeConfigID         = "<your-config-id>"
	accountID              = "<your-account-id>"
)

// 0=title, 1=image, 2=description, 3=sponsor, 4=CTA
const nativeRequest = `{"ver":"1.2","assets":[` +
	`{"id":0,"required":1,"title":{"len":80}},` +
	`{"id":1,"required":0,"img":{"type":3,"wmin":1,"hmin":1}},` +
	`{"id":2,"required":0,"data":{"type":2,"len":100}},` +
	`{"id":3,"required":0,"data":{"type":1}},` +
	`{"id":4,"required":0,"data":{"type":12,"len":30}}]}`

// Same shape as the web integration's window.tpc.data — one object per
// conversation, sent as-is under ext.prebid.bidder.tpc. UserID is a
// stable per-user identifier and SessionID a per-session one: generate
// and persist both yourself (e.g. a UUID per visitor, and a second UUID
// per conversation/session) — the web bundle auto-generates both when
// omitted, but there's no bundle here to do that for you. Messages grows
// by one entry each turn; the assistant entry is only added once a reply
// exists. Our server resolves this into whatever demand is actually
// configured for the placement — there's no partner-specific mapping to
// do on your side.
type message struct {
	Role    string `json:"role"`
	Content string `json:"content"`
}

type tpcData struct {
	AccountID string
	UserID    string
	SessionID string
	Messages  []message
}

func main() {
	data := tpcData{
		AccountID: accountID,
		UserID:    "stable-per-user-id",
		SessionID: "stable-per-session-id",
		Messages: []message{
			{Role: "user", Content: "I'm looking for new shoes"},
			// {Role: "assistant", Content: "..."} — add once your reply exists
		},
	}

	requestBody := map[string]any{
		"id":   fmt.Sprintf("req-%d", time.Now().UnixMilli()),
		"site": map[string]any{"page": "https://your-site.example.com/article"},
		"imp": []map[string]any{
			{
				"id":     "1",
				"native": map[string]any{"request": nativeRequest, "ver": "1.2"},
				"ext": map[string]any{
					"prebid": map[string]any{
						"storedrequest": map[string]any{"id": nativeConfigID},
						"bidder": map[string]any{"tpc": map[string]any{
							"accountId": data.AccountID,
							"userId":    data.UserID,
							"sessionId": data.SessionID,
							"messages":  data.Messages,
						}},
					},
				},
			},
		},
		"ext":  map[string]any{"prebid": map[string]any{"storedrequest": map[string]any{"id": auctionStoredRequestID}}},
		"regs": map[string]any{"ext": map[string]any{"gdpr": 0}},
		"user": map[string]any{"ext": map[string]any{"consent": ""}},
	}

	payload, _ := json.Marshal(requestBody)
	resp, err := http.Post(auctionURL, "application/json", bytes.NewReader(payload))
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()
	body, _ := io.ReadAll(resp.Body)

	var parsed map[string]any
	json.Unmarshal(body, &parsed)

	seatbid, _ := parsed["seatbid"].([]any)
	if len(seatbid) == 0 {
		fmt.Println("No bid returned.")
		return
	}
	seat := seatbid[0].(map[string]any)
	bid := seat["bid"].([]any)[0].(map[string]any)
	fmt.Println("Winning bid price:", bid["price"])
	fmt.Println("adm:", bid["adm"])
}
```

**Java** (JDK 11+, built-in `HttpClient`, no external dependency):

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;

public class Example {

    private static final String AUCTION_URL = "https://pbs.tpcsrv.com/openrtb2/auction";
    private static final String AUCTION_STORED_REQUEST_ID = "<your-auction-stored-request-id>";
    private static final String NATIVE_CONFIG_ID = "<your-config-id>";
    private static final String ACCOUNT_ID = "<your-account-id>";

    // 0=title, 1=image, 2=description, 3=sponsor, 4=CTA
    private static final String NATIVE_REQUEST = "{\"ver\":\"1.2\",\"assets\":["
            + "{\"id\":0,\"required\":1,\"title\":{\"len\":80}},"
            + "{\"id\":1,\"required\":0,\"img\":{\"type\":3,\"wmin\":1,\"hmin\":1}},"
            + "{\"id\":2,\"required\":0,\"data\":{\"type\":2,\"len\":100}},"
            + "{\"id\":3,\"required\":0,\"data\":{\"type\":1}},"
            + "{\"id\":4,\"required\":0,\"data\":{\"type\":12,\"len\":30}}]}";

    // Same shape as the web integration's window.tpc.data — one
    // conversation, sent as-is under ext.prebid.bidder.tpc. USER_ID is a
    // stable per-user identifier and SESSION_ID a per-session one: generate
    // and persist both yourself (e.g. a UUID per visitor, and a second UUID
    // per conversation/session) — the web bundle auto-generates both when
    // omitted, but there's no bundle here to do that for you. Our server
    // resolves this into whatever demand is actually configured for the
    // placement — there's no partner-specific mapping to do on your side.
    private static final String USER_ID = "stable-per-user-id";
    private static final String SESSION_ID = "stable-per-session-id";
    private static final String LAST_USER_MESSAGE = "I'm looking for new shoes";

    private static String tpcBidderExt() {
        return "\"bidder\":{\"tpc\":{"
                + "\"accountId\":" + jsonString(ACCOUNT_ID)
                + ",\"userId\":" + jsonString(USER_ID)
                + ",\"sessionId\":" + jsonString(SESSION_ID)
                + ",\"messages\":[{\"role\":\"user\",\"content\":" + jsonString(LAST_USER_MESSAGE) + "}]"
                + "}}";
    }

    public static void main(String[] args) throws Exception {
        String requestBody = "{"
                + "\"id\":\"req-" + System.currentTimeMillis() + "\","
                + "\"site\":{\"page\":\"https://your-site.example.com/article\"},"
                + "\"imp\":[{"
                + "\"id\":\"1\","
                + "\"native\":{\"request\":" + jsonString(NATIVE_REQUEST) + ",\"ver\":\"1.2\"},"
                + "\"ext\":{\"prebid\":{"
                + "\"storedrequest\":{\"id\":\"" + NATIVE_CONFIG_ID + "\"},"
                + tpcBidderExt()
                + "}}"
                + "}],"
                + "\"ext\":{\"prebid\":{\"storedrequest\":{\"id\":\"" + AUCTION_STORED_REQUEST_ID + "\"}}},"
                + "\"regs\":{\"ext\":{\"gdpr\":0}},"
                + "\"user\":{\"ext\":{\"consent\":\"\"}}"
                + "}";

        HttpClient client = HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(10))
                .build();
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(AUCTION_URL))
                .header("Content-Type", "application/json")
                .POST(HttpRequest.BodyPublishers.ofString(requestBody))
                .build();

        HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
        System.out.println("HTTP status: " + response.statusCode());
        System.out.println(response.body());
    }

    private static String jsonString(String s) {
        return "\"" + s.replace("\\", "\\\\").replace("\"", "\\\"") + "\"";
    }
}
```

A banner placement's request is simpler — swap the `native` object for
`"banner": {"w": 300, "h": 250}` and drop the `bidder` key entirely (only
native placements use per-auction content today).

### Contextual / chat-native ads

If your site or app is an AI assistant or chat interface, this is the
single biggest lever on ad relevance and yield — the server-side
counterpart of the web `<script>` integration's `window.tpc.data`. Both
integrations start from the same shape, sent as one object under
`ext.prebid.bidder.tpc`:

| Field | Purpose |
|---|---|
| `accountId` | Your Hola AI account ID |
| `userId` | A stable per-user identifier |
| `sessionId` | A stable per-session/per-conversation identifier |
| `messages` | The growing back-and-forth: `{ role: 'user' \| 'assistant', content }` |

Send the object as-is — there's no per-field mapping to do yourself. Our
server resolves it into whatever real demand is actually configured for
the placement; a placement with no content-driven demand on it simply
ignores the block. Two things that matter for getting full value out of
it:

- **`userId`/`sessionId` need to be genuinely stable, and you own
  generating them.** Unlike the web bundle, which auto-generates and
  persists both when omitted, there's no bundle here to do that for you —
  generate your own (e.g. a UUID per visitor for `userId`, a second UUID
  per conversation for `sessionId`) and keep sending the same values for
  the same person/session across auctions.
- **Send `messages` as it grows, turn by turn.** Your ad slot's auction
  typically fires as soon as the user submits a prompt, before your
  assistant has replied — that's fine, send whatever the conversation
  looks like at auction time. Re-run the auction with the assistant's
  reply added once it lands if you want a later impression to reflect it.

Omit the `bidder` key entirely for a placement that has no content-driven
demand — there's no reason to build one you don't need.

There's no server-side equivalent of the web guide's `hashedEmail` field
today — ask your account manager if you need logged-in-user matching for
a server-side integration.

### 2. Render the winning bid

Rendering happens in the visitor's browser, wherever that is relative to
your backend — you're responsible for getting the bid from step 1 to the
page (embedded in the initial HTML/JSON response, your own API, etc). Two
complete functions, one per format:

```javascript
// bid.adm is an HTML string for banner — never insert it directly into
// your page's DOM; always render it inside a sandboxed <iframe> so the
// creative can't touch your page's own DOM or cookies.
function renderBanner(bid, container) {
  const iframe = document.createElement('iframe');
  iframe.sandbox = 'allow-scripts allow-popups allow-popups-to-escape-sandbox';
  iframe.style.border = '0';
  iframe.style.width = (bid.w || 300) + 'px';
  iframe.style.height = (bid.h || 250) + 'px';
  iframe.srcdoc = bid.adm;
  container.appendChild(iframe);
}

// bid.adm is a JSON *string* for native — parse it, then map assets[].id
// to the same schema the request declared: 0=title, 1=image,
// 2=description, 3=sponsor, 4=CTA.
function renderNative(bid, container) {
  const native = JSON.parse(bid.adm);
  const assetsById = {};
  for (const asset of native.assets) assetsById[asset.id] = asset;

  const title = assetsById[0]?.title?.text;
  const image = assetsById[1]?.img?.url;
  const description = assetsById[2]?.data?.value;
  const sponsor = assetsById[3]?.data?.value;
  const cta = assetsById[4]?.data?.value;

  const root = document.createElement('div');
  root.className = 'hola-native-ad';
  root.innerHTML = `
    <img src="${image}" alt="${title || ''}">
    <div class="hola-native-title">${title || ''}</div>
    <div class="hola-native-desc">${description || ''}</div>
    <div class="hola-native-sponsor">${sponsor || ''}</div>
    <button class="hola-native-cta">${cta || 'Learn more'}</button>
  `;

  root.addEventListener('click', () => {
    if (native.link?.url) window.open(native.link.url, '_blank', 'noopener');
    fireTrackers(native.link?.clicktrackers);
  });

  container.appendChild(root);
  fireTrackers(native.imptrackers);
}

function fireTrackers(urls) {
  if (!urls) return;
  for (const url of urls) new Image().src = url;
}
```

Both functions fire tracking pixels for you (`imptrackers` on render,
`clicktrackers` on click) — you don't need Prebid.js or any other library
on the page for this integration path; these two functions are the whole
rendering layer.

## Ad formats

Banner (display) and native (sponsored card — title/image/description/
sponsor/CTA, the 5-asset schema shown above) are supported. Video is not
yet available for server-side; ask your account manager if you need it.

## Timeouts

Our Prebid Server's own auction budget is **2500ms** — it queries every
demand partner on the imp in parallel and returns whatever bids have come
back by then. Set your own HTTP client timeout comfortably above that, so
you're not cutting the auction off early: **3500ms or more** is a safe
floor (the same failsafe margin the web bundle uses). A slow demand
partner costs you that partner's bid, not the whole auction — the
response still comes back with whatever did bid in time.

## Refreshing ads

The web bundle shows up to `adMaxRefresh` sequential ads per session (3 by
default) with a pause between them. There's no equivalent built into the
Prebid Server endpoint itself — a "refresh" from a server-side integration
is just running the auction request again with a new `id`, on whatever
cadence and cap fits your product (e.g. only after the current ad's
`adDisplaySeconds`-equivalent has elapsed on your side, and stopping after
your own per-session cap). You own that pacing logic entirely.

## Consent

Set these fields yourself from your own consent state (CMP, cookie banner,
server-side geo/consent logic — whatever you use):

| Field | Purpose |
|---|---|
| `regs.ext.gdpr` | `1` if GDPR applies, `0` otherwise |
| `user.ext.consent` | TCF 2.x consent string |
| `regs.us_privacy` | US state privacy string (CCPA etc.) |
| `regs.gpp` / `regs.gpp_sid` | Global Privacy Platform string + applicable section IDs |

If you omit these, our server applies a default "no restrictions" posture
— useful for getting started, but you should pass your own real consent
signals once your consent flow is in place.

For testing only — illustrative TCF 2.3 and GPP (USNAT, section 7)
strings, the same ones from the web integration guide:

| | String |
|---|---|
| TCF — all purposes/vendors consented | `CQJz4oAQJz4oAGXABBENBdFsAP_gAEPgAAATIIDoBJCoAAAAAA` |
| TCF — no consent | `CQAAAAAAAAAAAAAAAABzAAAAAAAAAAAAAAAAAAAAAAAAAAAA` |
| GPP — all sharing permitted | `DBABBg==` (pass `regs.gpp_sid: [7]` alongside it) |
| GPP — all opt-outs set | `DBABjw==` (pass `regs.gpp_sid: [7]` alongside it) |

Generate real strings with your CMP or a TCF/GPP-compliant encoder library
such as [@iabtcf/encoder](https://www.npmjs.com/package/@iabtcf/encoder)
for production use — these are for testing only.

## Reporting API

Want your revenue data programmatically? See the
[Reporting API Guide](/public-docs/reporting-api/) — a simple REST API
authenticated with an API key your account manager can issue you, covering
the same summary/timeseries/breakdown data shown in the dashboard. It's
the same API regardless of which integration path (web, mobile SDK, or
this one) drives the underlying traffic.

## Troubleshooting

### No bid / empty `seatbid`

A missing `seatbid` array is often a genuine no-fill, not an error — check
`ext.errors` in the response first. No demand source rejects the whole
auction over missing or incomplete content — each one fails soft, scoped
to its own bid only, and the rest of the auction proceeds normally
(`ext.errors.<source>` names which one). Sending `userId`/`sessionId`/
`messages` consistently, as described in "Contextual / chat-native ads"
above, gets you the best match rate across whichever demand is configured
for the placement — you don't need to reason about which source wants
which field.

### `Stored Request with ID=... not found` / `Stored Imp with ID=... not
found`

Your account ID, auction stored request ID, or config ID doesn't match
what your account manager gave you — double check for typos, and confirm
you're using `"site"`, not `"app"`, in your request (this guide's config
IDs are provisioned for a web-style `site` object, not the Mobile SDK's
`app` object).

### Bid received but nothing renders

Confirm `bid.adm` is being parsed as JSON for native (`JSON.parse`, not
inserted as raw HTML) and rendered inside a sandboxed `<iframe>` for
banner — mixing the two up is the most common integration bug.

## Contact

For integration questions, placement provisioning, or to request additional
formats, contact your Hola AI account manager.
