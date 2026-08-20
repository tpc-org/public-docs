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
account's auction stored request ID, and (for Thrad-backed native
placements) a per-user identifier.

**Node.js** (no dependencies — uses the built-in `fetch`):

```javascript
const AUCTION_URL = 'https://pbs.tpcsrv.com/openrtb2/auction';
const AUCTION_STORED_REQUEST_ID = '<your-auction-stored-request-id>';
const NATIVE_CONFIG_ID = '<your-config-id>';

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
            bidder: { thrad: { userId: 'stable-per-user-id' } },
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
)

// 0=title, 1=image, 2=description, 3=sponsor, 4=CTA
const nativeRequest = `{"ver":"1.2","assets":[` +
	`{"id":0,"required":1,"title":{"len":80}},` +
	`{"id":1,"required":0,"img":{"type":3,"wmin":1,"hmin":1}},` +
	`{"id":2,"required":0,"data":{"type":2,"len":100}},` +
	`{"id":3,"required":0,"data":{"type":1}},` +
	`{"id":4,"required":0,"data":{"type":12,"len":30}}]}`

func main() {
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
						"bidder": map[string]any{
							"thrad": map[string]any{"userId": "stable-per-user-id"},
						},
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

    // 0=title, 1=image, 2=description, 3=sponsor, 4=CTA
    private static final String NATIVE_REQUEST = "{\"ver\":\"1.2\",\"assets\":["
            + "{\"id\":0,\"required\":1,\"title\":{\"len\":80}},"
            + "{\"id\":1,\"required\":0,\"img\":{\"type\":3,\"wmin\":1,\"hmin\":1}},"
            + "{\"id\":2,\"required\":0,\"data\":{\"type\":2,\"len\":100}},"
            + "{\"id\":3,\"required\":0,\"data\":{\"type\":1}},"
            + "{\"id\":4,\"required\":0,\"data\":{\"type\":12,\"len\":30}}]}";

    public static void main(String[] args) throws Exception {
        String requestBody = "{"
                + "\"id\":\"req-" + System.currentTimeMillis() + "\","
                + "\"site\":{\"page\":\"https://your-site.example.com/article\"},"
                + "\"imp\":[{"
                + "\"id\":\"1\","
                + "\"native\":{\"request\":" + jsonString(NATIVE_REQUEST) + ",\"ver\":\"1.2\"},"
                + "\"ext\":{\"prebid\":{"
                + "\"storedrequest\":{\"id\":\"" + NATIVE_CONFIG_ID + "\"},"
                + "\"bidder\":{\"thrad\":{\"userId\":\"stable-per-user-id\"}}"
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
`"banner": {"w": 300, "h": 250}` and drop the `thrad` userId (only native
placements route through Thrad today).

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

## Troubleshooting

### No bid / empty `seatbid`

A missing `seatbid` array is often a genuine no-fill, not an error — check
`ext.errors` in the response first. A common cause for native placements:
Thrad requires `ext.prebid.bidder.thrad.userId` on the imp; PBS returns a
400 rejecting the whole imp if it's missing.

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
