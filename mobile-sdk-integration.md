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

## Dynamic per-auction content (Thrad / Imprezia)

If your app is an AI assistant or chat interface, this is the single
biggest lever on ad relevance and yield for a native placement — the
in-app counterpart of the web `<script>` integration's `window.tpc.data`
and the [Server-side Guide](/public-docs/server-side-integration/)'s
`buildBidderExt()` helper. All three integrations start from the same
shape: one object per conversation, with `userId` (a stable per-user
identifier), `sessionId` (a stable per-session identifier), and
`messages` (the growing back-and-forth, `{role: 'user' | 'assistant',
content}`). Where they differ is only in *how* that object reaches PBS:
the web bundle and server-side backend build the whole request
themselves; here, the SDK builds the request for you, so you attach your
own JSON fragment to the specific ad unit before calling `fetchDemand`/
`loadAd`, using the SDK's own imp-level ORTB merge method:

- iOS: `adUnit.setImpORTBConfig(jsonString)`
- Android: `adUnit.setImpOrtbConfig(jsonString)`

Both merge the JSON you pass into that ad unit's own `imp` object,
documented by Prebid as an imp-level merge (not a replace) with the
Stored Config's server-side fields. **We have not verified this merge's
exact behavior against a running app** — no iOS/Android app exists in
this workspace to test against, only Prebid's own published docs. Treat
it as unconfirmed until you validate it yourself with the debug procedure
below, before relying on it in production.

| Partner | Reads | Derived from |
|---|---|---|
| Thrad | `ext.prebid.bidder.thrad.userId` — identity | `userId` |
| Imprezia | `ext.prebid.bidder.imprezia.request` / `.response` — content, `.sessionId` — identity | the latest `user` / `assistant` entries in `messages`, plus `sessionId` |

**iOS (Swift):**

```swift
// Same shape as the web/server-side integration's shared conversation
// object — userId, sessionId, and messages. userId is a stable per-user
// identifier and sessionId a per-session one: generate and persist both
// yourself (e.g. a UUID per visitor, and a second UUID per conversation)
// — there's no bundle or backend helper to do this for you here.
// Imprezia's real API requires sessionId even though our own schema
// marks it optional — omitting it gets a live 400 from Imprezia, not a
// graceful skip.
struct Message { let role: String; let content: String }

// Derives each native demand partner's per-auction bidder ext from one
// shared conversation object: Thrad reads identity (userId), Imprezia
// reads content (the latest user/assistant turn) plus sessionId. Skip
// this call entirely (don't call setImpORTBConfig) for a placement not
// provisioned with either partner.
func buildBidderExt(userId: String, sessionId: String, messages: [Message]) -> String {
    let lastUser = messages.last(where: { $0.role == "user" })?.content ?? ""
    let lastAssistant = messages.last(where: { $0.role == "assistant" })?.content
    let response = (lastAssistant?.isEmpty == false) ? lastAssistant! : " " // ' ' placeholder — never ""

    let impExt: [String: Any] = [
        "ext": ["prebid": ["bidder": [
            "thrad": ["userId": userId],
            "imprezia": ["request": lastUser, "response": response, "sessionId": sessionId],
        ]]]
    ]
    let data = try! JSONSerialization.data(withJSONObject: impExt)
    return String(data: data, encoding: .utf8)!
}

let nativeAdUnit = NativeRequestAdUnit(configId: "<your-config-id>")
nativeAdUnit.setImpORTBConfig(buildBidderExt(
    userId: "stable-per-user-id",
    sessionId: "stable-per-session-id",
    messages: [Message(role: "user", content: "I'm looking for new shoes")]
    // Add Message(role: "assistant", content: "...") once your reply
    // exists, and call buildBidderExt + setImpORTBConfig again before
    // the next auction — same "re-run on every new turn" pattern as
    // window.tpc.requestAd on web.
))
nativeAdUnit.fetchDemand { [weak self] result, kvResultDict in
    // render using kvResultDict / result
}
```

**Android (Kotlin):**

```kotlin
// Same shape as the web/server-side integration's shared conversation
// object — userId, sessionId, and messages. userId is a stable per-user
// identifier and sessionId a per-session one: generate and persist both
// yourself — there's no bundle or backend helper to do this for you
// here. Imprezia's real API requires sessionId even though our own
// schema marks it optional — omitting it gets a live 400 from Imprezia,
// not a graceful skip.
data class Message(val role: String, val content: String)

// Derives each native demand partner's per-auction bidder ext from one
// shared conversation object: Thrad reads identity (userId), Imprezia
// reads content (the latest user/assistant turn) plus sessionId. Skip
// this call entirely (don't call setImpOrtbConfig) for a placement not
// provisioned with either partner.
fun buildBidderExt(userId: String, sessionId: String, messages: List<Message>): String {
    val lastUser = messages.lastOrNull { it.role == "user" }?.content ?: ""
    val lastAssistant = messages.lastOrNull { it.role == "assistant" }?.content
    val response = if (!lastAssistant.isNullOrEmpty()) lastAssistant else " " // ' ' placeholder — never ""

    val bidder = JSONObject()
        .put("thrad", JSONObject().put("userId", userId))
        .put("imprezia", JSONObject()
            .put("request", lastUser)
            .put("response", response)
            .put("sessionId", sessionId))
    return JSONObject().put("ext", JSONObject().put("prebid", JSONObject().put("bidder", bidder))).toString()
}

val nativeAdUnit = NativeAdUnit("<your-config-id>")
nativeAdUnit.setImpOrtbConfig(buildBidderExt(
    userId = "stable-per-user-id",
    sessionId = "stable-per-session-id",
    messages = listOf(Message(role = "user", content = "I'm looking for new shoes"))
    // Add Message(role = "assistant", content = "...") once your reply
    // exists, and call buildBidderExt + setImpOrtbConfig again before
    // the next auction — same "re-run on every new turn" pattern as
    // window.tpc.requestAd on web.
))
nativeAdUnit.fetchDemand { result ->
    // render using the returned native ad
}
```

Three things that will bite you if you build your own version of this:

- **Never send Imprezia's `response` as an empty string.** Your ad unit's
  auction typically fires as soon as the user submits a prompt, before
  your assistant has replied — so there's no assistant message yet.
  Imprezia's API rejects an empty string outright (`"response" is not
  allowed to be empty`); both samples above fall back to a single space
  (`' '`) whenever there's no assistant turn.
- **Always send Imprezia's `sessionId`.** Our own schema marks it
  optional, but Imprezia's real API requires it — omit it and every
  Imprezia auction gets a live 400 (`"sessionId" is required`), not a
  graceful skip. Same rule as the server-side guide's own correction of
  this exact gap.
- **Call `setImpORTBConfig`/`setImpOrtbConfig` again on every new
  auction**, including refreshes — it's set on the ad unit instance, not
  persisted globally, so a stale call from an earlier turn will keep
  sending stale context otherwise.

### Validating the merge landed correctly

Because the merge happens inside the SDK, on the device, there's no way to
see it from your app code alone — you're trusting Prebid's own
documentation that it lands where you expect. Confirm it directly against
PBS before shipping:

1. Fire a real auction from your app (or a `test:1` one) against your real
   config ID.
2. Separately, hand-build the equivalent request using the curl skeleton
   in Troubleshooting below, with `"test": 1` and `"ext": {"prebid":
   {"debug": true}}` added, and the same `thrad`/`imprezia` fields you're
   sending from the app.
3. Inspect `response.ext.debug.resolvedrequest.imp[0].ext.prebid.bidder.<bidder>`
   in the response — this is PBS's own post-merge, pre-adapter view of
   the imp. Confirm your dynamic fields (`userId`, `sessionId`,
   `request`/`response`) show up there alongside the Stored Config's
   static fields (`publisherId` for Thrad; `siteId`/`placementId` for
   Imprezia) — not nested somewhere else, and not overwriting the static
   fields instead of merging alongside them.
4. Treat this as a required step before calling any specific app
   integration "live," not an optional nicety — a merge landing at the
   wrong key fails exactly like Imprezia's params silently failing to
   reach PBS did on web earlier today, just via a different code path.

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
