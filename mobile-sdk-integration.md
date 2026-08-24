---
layout: page
---

# Hola AI Ads — Mobile SDK Integration Guide

**[Integration Guide](/public-docs/mobile-sdk-integration/)** · [Web Integration Guide](/public-docs/publisher-integration/) · [Server-side Guide](/public-docs/server-side-integration/) · [Reporting API Guide](/public-docs/reporting-api/) · [Payment Details Guide](/public-docs/payment-details/) · [Support](#contact)

---

This guide is for app developers integrating Hola AI Ads into a native iOS or
Android app via the [Prebid Mobile SDK](https://docs.prebid.org/prebid-mobile/prebid-mobile.html).
This is a separate integration path from the web `<script>` tag guide — the
Mobile SDK is a native library that talks to our Prebid Server directly over
OpenRTB, without a browser or JavaScript bundle involved at all.

You'll need: the Prebid Mobile SDK (iOS or Android), our Prebid Server
endpoint, your account ID, and one config ID per ad placement. Your Hola AI
account manager provides the account ID and config IDs.

## Which integration path

Prebid Mobile SDK supports several integration modes (GAM bidding-only, GAM
Prebid-rendered, mediation adapters). **Hola AI uses the "no ad server" /
Prebid-Rendered path** — the SDK renders the winning bid directly, the same
way our web bundle renders creatives itself without a Google Ad Manager or
mediation layer in between.

## Quick start

### 1. Initialize the SDK

**iOS (Swift):**

```swift
Prebid.initializeSDK("https://pbs.tpcsrv.com/openrtb2/auction") { status, error in
    // handle initialization result
}
Prebid.shared.prebidServerAccountId = "<your-account-id>"
```

**Android (Kotlin):**

```kotlin
PrebidMobile.setPrebidServerAccountId("<your-account-id>")
PrebidMobile.initializeSdk(requireContext(), "https://pbs.tpcsrv.com/openrtb2/auction") { status ->
    // handle initialization result
}
```

Do this once, at app launch.

### 2. Define an ad unit per placement

Each placement your account manager gives you is a **config ID** — the
Prebid Server Stored Impression ID for that placement. You don't configure
demand partners yourself; that's all resolved server-side from the config
ID, exactly like the web bundle.

**Banner (iOS):**

```swift
let banner = BannerView(frame: CGRect(origin: .zero, size: adSize),
                         configID: "<your-config-id>",
                         adSize: adSize)
banner.delegate = self
banner.loadAd()
```

**Banner (Android):**

```kotlin
bannerView = BannerView(requireContext(), "<your-config-id>", adSize)
bannerView?.setBannerListener(this)
viewContainer?.addView(bannerView)
bannerView?.loadAd()
```

**Native (iOS):**

```swift
let nativeAdUnit = NativeRequestAdUnit(configId: "<your-config-id>")
nativeAdUnit.fetchDemand { [weak self] result, kvResultDict in
    // render using kvResultDict / result
}
```

**Native (Android):**

```kotlin
val nativeAdUnit = NativeAdUnit("<your-config-id>")
nativeAdUnit.fetchDemand { result ->
    // render using the returned native ad
}
```

That's it. The SDK handles the auction request, our stored config resolves
the real bidder mix server-side, and the SDK renders the winning creative —
no per-bidder parameters, no ad-server line items to set up.

## Ad formats

Same two formats as the web bundle's core offering: banner (display) and
native (sponsored card with title/image/body/CTA — same 4-asset schema as
web). Video is not yet supported for in-app; ask your account manager if you
need it.

## Dynamic per-auction content

If your app is an AI assistant or chat interface, this is the single
biggest lever on ad relevance and yield for a native placement — the
in-app counterpart of the web `<script>` integration's `window.tpc.data`
and the [Server-side Guide](/public-docs/server-side-integration/)'s
generic content object. All three integrations start from the same
shape, sent as-is under `ext.prebid.bidder.tpc`: one object per
conversation, with `accountId`, `userId` (a stable per-user identifier),
`sessionId` (a stable per-session identifier), `messages` (the growing
back-and-forth, `{role: 'user' | 'assistant', content}`), and optionally
`deviceContext` (`{deviceType, viewportWidth, viewportHeight}` —
`deviceType` is `desktop` | `mobile` | `tablet`, viewports are CSS pixels
of the area the ad renders into; include it if you know it, some demand
uses it to better match device signals — omitting it never breaks a
placement that doesn't need it). Our server resolves that object into
whatever real demand is actually configured for the placement — you
never send anything demand-source-specific yourself. Where the
integrations differ is only in *how* that
object reaches PBS: the web bundle and server-side backend build the
whole request themselves; here, the SDK builds the request for you, so
you attach your own JSON fragment to the specific ad unit before calling
`fetchDemand`/`loadAd`, using the SDK's own imp-level ORTB merge method:

- iOS: `adUnit.setImpORTBConfig(jsonString)`
- Android: `adUnit.setImpOrtbConfig(jsonString)`

Both merge the JSON you pass into that ad unit's own `imp` object,
documented by Prebid as an imp-level merge (not a replace) with the
Stored Config's server-side fields. **We have not verified this merge's
exact behavior against a running app** — no iOS/Android app exists in
this workspace to test against, only Prebid's own published docs. Treat
it as unconfirmed until you validate it yourself with the debug procedure
below, before relying on it in production. (Server-side resolution of the
generic block itself — the part PBS does after the merge — is verified
live; see this guide's internal section and
`server-side-integration.md`'s internal "Platform mechanics" section.)

**iOS (Swift):**

```swift
// Same shape as the web/server-side integration's shared conversation
// object — accountId, userId, sessionId, and messages, sent as-is under
// ext.prebid.bidder.tpc. userId is a stable per-user identifier and
// sessionId a per-session one: generate and persist both yourself (e.g.
// a UUID per visitor, and a second UUID per conversation) — there's no
// bundle or backend helper to do this for you here. Our server resolves
// this into whatever demand is actually configured for the placement —
// there's no partner-specific mapping to do on your side.
struct Message { let role: String; let content: String }
struct DeviceContext { let deviceType: String; let viewportWidth: Int; let viewportHeight: Int }

func tpcBidderExt(accountId: String, userId: String, sessionId: String, messages: [Message], deviceContext: DeviceContext? = nil) -> String {
    var tpc: [String: Any] = [
        "accountId": accountId,
        "userId": userId,
        "sessionId": sessionId,
        "messages": messages.map { ["role": $0.role, "content": $0.content] },
    ]
    // Optional — some demand uses it to better match device signals.
    // Only include it if you actually know the real device/viewport shape;
    // never guess.
    if let dc = deviceContext {
        tpc["deviceContext"] = ["deviceType": dc.deviceType, "viewportWidth": dc.viewportWidth, "viewportHeight": dc.viewportHeight]
    }
    let impExt: [String: Any] = ["ext": ["prebid": ["bidder": ["tpc": tpc]]]]
    let data = try! JSONSerialization.data(withJSONObject: impExt)
    return String(data: data, encoding: .utf8)!
}

let nativeAdUnit = NativeRequestAdUnit(configId: "<your-config-id>")
nativeAdUnit.setImpORTBConfig(tpcBidderExt(
    accountId: "<your-account-id>",
    userId: "stable-per-user-id",
    sessionId: "stable-per-session-id",
    messages: [Message(role: "user", content: "I'm looking for new shoes")]
    // Add Message(role: "assistant", content: "...") once your reply
    // exists, and call tpcBidderExt + setImpORTBConfig again before
    // the next auction — same "re-run on every new turn" pattern as
    // window.tpc.requestAd on web.
))
nativeAdUnit.fetchDemand { [weak self] result, kvResultDict in
    // render using kvResultDict / result, then report measurement — see
    // "Reporting required measurement" below.
}
```

**Android (Kotlin):**

```kotlin
// Same shape as the web/server-side integration's shared conversation
// object — accountId, userId, sessionId, and messages, sent as-is under
// ext.prebid.bidder.tpc. userId is a stable per-user identifier and
// sessionId a per-session one: generate and persist both yourself —
// there's no bundle or backend helper to do this for you here. Our
// server resolves this into whatever demand is actually configured for
// the placement — there's no partner-specific mapping to do on your side.
data class Message(val role: String, val content: String)
data class DeviceContext(val deviceType: String, val viewportWidth: Int, val viewportHeight: Int)

fun tpcBidderExt(accountId: String, userId: String, sessionId: String, messages: List<Message>, deviceContext: DeviceContext? = null): String {
    val tpc = JSONObject()
        .put("accountId", accountId)
        .put("userId", userId)
        .put("sessionId", sessionId)
        .put("messages", messages.map {
            JSONObject().put("role", it.role).put("content", it.content)
        })
    // Optional — some demand uses it to better match device signals. Only
    // include it if you actually know the real device/viewport shape;
    // never guess.
    deviceContext?.let {
        tpc.put("deviceContext", JSONObject()
            .put("deviceType", it.deviceType)
            .put("viewportWidth", it.viewportWidth)
            .put("viewportHeight", it.viewportHeight))
    }
    val bidder = JSONObject().put("tpc", tpc)
    return JSONObject().put("ext", JSONObject().put("prebid", JSONObject().put("bidder", bidder))).toString()
}

val nativeAdUnit = NativeAdUnit("<your-config-id>")
nativeAdUnit.setImpOrtbConfig(tpcBidderExt(
    accountId = "<your-account-id>",
    userId = "stable-per-user-id",
    sessionId = "stable-per-session-id",
    messages = listOf(Message(role = "user", content = "I'm looking for new shoes"))
    // Add Message(role = "assistant", content = "...") once your reply
    // exists, and call tpcBidderExt + setImpOrtbConfig again before
    // the next auction — same "re-run on every new turn" pattern as
    // window.tpc.requestAd on web.
))
nativeAdUnit.fetchDemand { result ->
    // render using the returned native ad, then report measurement — see
    // "Reporting required measurement" below.
}
```

Two things that will bite you if you build your own version of this:

- **Send `messages` as it grows, turn by turn.** Your ad unit's auction
  typically fires as soon as the user submits a prompt, before your
  assistant has replied — that's fine, send whatever the conversation
  looks like at auction time; our server never needs an empty field
  worked around on your side.
- **Call `setImpORTBConfig`/`setImpOrtbConfig` again on every new
  auction**, including refreshes — it's set on the ad unit instance, not
  persisted globally, so a stale call from an earlier turn will keep
  sending stale context otherwise.

### Reporting required measurement

Some demand carries measurement beyond what the SDK's own click/impression
handling covers — inspect the raw native response for an `eventtrackers`
array (`{event, method, url}` — event 1 fires once at insertion, event 2
fires once at MRC-standard viewability: >=50% of the creative visible for
1 continuous second) and a top-level `ext` object whose values may include
their own `beaconBaseUrl`. Measurement can also arrive via a frame URL
instead of the array (`ext.<partner>.impressionFrameUrl`/
`viewabilityFrameUrl`) — the two channels are mutually exclusive per
serve, so check for the frame field first and only fall back to the
array when it's absent; a frame URL must be embedded (e.g. a hidden
WebView), never fetched directly, since the pixel only counts when
actually rendered as a document. All of this is optional and additive —
a bid without any of it needs nothing extra, and this code never assumes
which (if any) demand behind a placement uses them. **This section is unverified** — the
Prebid Mobile SDK's own native ad result type is not confirmed to expose
the raw `ext`/`eventtrackers` fields from the underlying OpenRTB response
(no app exists in this workspace to check against); treat the sketch
below as a starting point, not confirmed-correct code, until validated
against a real SDK response — see "Validating the merge landed correctly"
below for the same debug procedure applied to the response side instead
of the request side.

**iOS (Swift), sketch:**

```swift
// Never load a frame URL via URLSession — it only counts when actually
// rendered as a document. Use a hidden WKWebView (1x1, off-screen) so
// the pixel fires the same way a browser's iframe would; a plain data
// task would just fetch the bytes without "showing" it, which some
// measurement vendors don't count as delivered.
func embedTrackerFrame(url: String) {
    let webView = WKWebView(frame: CGRect(x: -1000, y: -1000, width: 1, height: 1))
    webView.load(URLRequest(url: URL(string: url)!))
    // Retain webView (e.g. in an instance array) until its didFinish
    // delegate callback fires, then release it — omitted here for brevity.
}

func deliverExtraMeasurement(native: [String: Any], eventCode: Int) {
    guard let ext = native["ext"] as? [String: Any],
          let beaconMeta = ext.values.compactMap({ $0 as? [String: Any] }).first(where: { $0["beaconBaseUrl"] != nil })
    else {
        // No beacon metadata at all -- still fire any plain array trackers.
        let eventTrackers = native["eventtrackers"] as? [[String: Any]] ?? []
        let urls = eventTrackers.filter { ($0["event"] as? Int) == eventCode }.compactMap { $0["url"] as? String }
        for url in urls { URLSession.shared.dataTask(with: URL(string: url)!).resume() }
        return
    }

    // Two mutually-exclusive channels depending on the serve: a frame URL
    // (impressionFrameUrl/viewabilityFrameUrl) that must be embedded, or
    // the plain eventtrackers array. Check the frame field first.
    let frameKey = eventCode == 1 ? "impressionFrameUrl" : "viewabilityFrameUrl"
    if let frameUrl = beaconMeta[frameKey] as? String {
        embedTrackerFrame(url: frameUrl)
    } else {
        let eventTrackers = native["eventtrackers"] as? [[String: Any]] ?? []
        let urls = eventTrackers.filter { ($0["event"] as? Int) == eventCode }.compactMap { $0["url"] as? String }
        for url in urls { URLSession.shared.dataTask(with: URL(string: url)!).resume() }
    }

    let eventType = eventCode == 1 ? "sdk_impression_inserted" : "sdk_impression"
    var payload: [String: Any] = [
        "eventId": "evt_\(Int(Date().timeIntervalSince1970 * 1000))_\(Int.random(in: 0...999999))",
        "eventType": eventType,
        "requestId": beaconMeta["requestId"] ?? "",
        "sdkVersion": "mobile-sdk-integration/1.0.0",
        "clientTimestamp": ISO8601DateFormatter().string(from: Date()),
        "serverTimestamp": beaconMeta["servedAt"] ?? "",
        "generatedUrl": (native["link"] as? [String: Any])?["url"] ?? "",
        "impressionUuid": beaconMeta["impressionUuid"] ?? "",
        "placementType": "uicard",
        "publisherId": beaconMeta["publisherId"] ?? "",
        "sessionId": beaconMeta["sessionId"] ?? "",
    ]
    if let token = beaconMeta["beaconToken"] as? [String: Any] {
        payload["impressionToken"] = token["token"]
        payload["tokenIssuedAt"] = token["issuedAt"]
        payload["tokenKid"] = token["kid"]
    }
    var request = URLRequest(url: URL(string: "\(beaconMeta["beaconBaseUrl"]!)/v1/events/sdk-impression")!)
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    request.httpBody = try! JSONSerialization.data(withJSONObject: payload)
    URLSession.shared.dataTask(with: request).resume()
}
// Call deliverExtraMeasurement(native:, eventCode: 1) once at insertion,
// and again with eventCode: 2 once you've independently confirmed >=50%
// viewability for 1 continuous second (the SDK may already track
// viewability for its own billing — reuse that signal if so, rather than
// building a second observer).
```

**Android (Kotlin), sketch:**

```kotlin
// Never fetch a frame URL as plain bytes -- it only counts when actually
// rendered as a document. A hidden, off-screen WebView (1x1) fires the
// pixel the same way a browser's iframe would.
fun embedTrackerFrame(context: Context, url: String) {
    val webView = WebView(context).apply {
        layoutParams = ViewGroup.LayoutParams(1, 1)
        loadUrl(url)
    }
    // Attach webView to an off-screen container view until its
    // WebViewClient.onPageFinished fires, then detach/destroy it --
    // omitted here for brevity.
}

fun deliverExtraMeasurement(native: JSONObject, eventCode: Int, context: Context) {
    fun fireArrayTrackers() {
        val eventTrackers = native.optJSONArray("eventtrackers")
        if (eventTrackers != null) {
            for (i in 0 until eventTrackers.length()) {
                val t = eventTrackers.getJSONObject(i)
                if (t.optInt("event") == eventCode) fireUrl(t.getString("url"))
            }
        }
    }

    val ext = native.optJSONObject("ext")
    var beaconMeta: JSONObject? = null
    ext?.keys()?.forEach { k -> if (ext.optJSONObject(k)?.has("beaconBaseUrl") == true) beaconMeta = ext.getJSONObject(k) }
    val meta = beaconMeta ?: run { fireArrayTrackers(); return }

    // Two mutually-exclusive channels depending on the serve: a frame URL
    // (impressionFrameUrl/viewabilityFrameUrl) that must be embedded, or
    // the plain eventtrackers array. Check the frame field first.
    val frameKey = if (eventCode == 1) "impressionFrameUrl" else "viewabilityFrameUrl"
    val frameUrl = meta.optString(frameKey).takeIf { it.isNotEmpty() }
    if (frameUrl != null) embedTrackerFrame(context, frameUrl) else fireArrayTrackers()

    val eventType = if (eventCode == 1) "sdk_impression_inserted" else "sdk_impression"
    val payload = JSONObject()
        .put("eventId", "evt_${System.currentTimeMillis()}_${(0..999999).random()}")
        .put("eventType", eventType)
        .put("requestId", meta.optString("requestId"))
        .put("sdkVersion", "mobile-sdk-integration/1.0.0")
        .put("clientTimestamp", java.time.Instant.now().toString())
        .put("serverTimestamp", meta.optString("servedAt"))
        .put("generatedUrl", native.optJSONObject("link")?.optString("url"))
        .put("impressionUuid", meta.optString("impressionUuid"))
        .put("placementType", "uicard")
        .put("publisherId", meta.optString("publisherId"))
        .put("sessionId", meta.optString("sessionId"))
    meta.optJSONObject("beaconToken")?.let { token ->
        payload.put("impressionToken", token.optString("token"))
            .put("tokenIssuedAt", token.optLong("issuedAt"))
            .put("tokenKid", token.optString("kid"))
    }
    postJson("${meta.getString("beaconBaseUrl")}/v1/events/sdk-impression", payload)
}
// Call deliverExtraMeasurement(native, eventCode = 1, context) once at
// insertion, and again with eventCode = 2 once you've independently
// confirmed >=50% viewability for 1 continuous second (the SDK may
// already track viewability for its own billing — reuse that signal if
// so, rather than building a second observer). fireUrl/postJson are your
// own thin GET/POST helpers, omitted here.
```

### Validating the merge landed correctly

Because the merge happens inside the SDK, on the device, there's no way to
see it from your app code alone — you're trusting Prebid's own
documentation that it lands where you expect. Confirm it directly against
PBS before shipping:

1. Fire a real auction from your app (or a `test:1` one) against your real
   config ID.
2. Separately, hand-build the equivalent request using the curl skeleton
   in Troubleshooting below, with `"test": 1` and `"ext": {"prebid":
   {"debug": true}}` added, and the same `ext.prebid.bidder.tpc` fields
   you're sending from the app.
3. Inspect `response.ext.debug.resolvedrequest.imp[0].ext.prebid.bidder`
   in the response — this is PBS's own post-merge, pre-adapter view of
   the imp. Confirm your `tpc` object (`accountId`, `userId`, `sessionId`,
   `messages`) shows up there, and that it's actually been resolved into
   whatever demand is configured for the placement.
4. Treat this as a required step before calling any specific app
   integration "live," not an optional nicety — a merge landing at the
   wrong key fails just as silently as a missed field anywhere else in
   this pipeline, just via a different code path.

## Consent

Our server has a default "no restrictions" consent posture (same spirit as
the web bundle's default), which is used **only when your app doesn't supply
its own consent signals**. If your app has its own CMP/consent SDK (for GDPR
TCF or US state privacy/GPP), pass those values through the Prebid Mobile
SDK's global privacy parameters — real values you provide always take
precedence over our defaults:

**iOS:**

```swift
Prebid.shared.gdprApplies = true
Prebid.shared.purposeConsents = "<tcf-2.x-consent-string>"
```

**Android:**

```kotlin
PrebidMobile.setSubjectToGDPR(true)
PrebidMobile.setPurposeConsents("<tcf-2.x-consent-string>")
```

If your app doesn't have its own consent management, you don't need to set
anything — the default posture applies automatically.

## A note on demand partner attribution

On web, our bundle rewrites every bid's displayed bidder code to a unified
`tpc` label for clean targeting-key naming (the real demand partner is
preserved separately for analytics). That rewrite is JavaScript logic with
no native equivalent, so **in the Mobile SDK you may see the underlying
demand source's name** directly in the bid response rather than a unified
label. This is expected — not a bug — and doesn't affect rendering or
revenue.

## Performance notes

- `loadAd()` must be called on the main thread (both platforms).
- Default bid timeout matches our web bundle: keep your app's own timeout
  expectations around 1.5–2s per auction.

## Troubleshooting

### No ads appearing

1. Confirm `Prebid.initializeSDK`/`PrebidMobile.initializeSdk` completed
   successfully before loading any ad unit.
2. Confirm the config ID matches your assigned placement exactly.
3. Check your app's network logs for a POST to
   `https://pbs.tpcsrv.com/openrtb2/auction` — if missing, the SDK didn't
   fire an auction request.

### Bid received but creative doesn't render

Confirm the ad view/ad unit's delegate/listener is set before calling
`loadAd()` — rendering happens via callback, not synchronously.

## Contact

For integration questions, placement provisioning, or to request additional
formats, contact your Hola AI account manager.
