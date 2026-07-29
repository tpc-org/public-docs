---
layout: page
---

# Hola AI Ads — Mobile SDK Integration Guide

**[Integration Guide](/public-docs/mobile-sdk-integration/)** · [Web Integration Guide](/public-docs/publisher-integration/) · [Reporting API Guide](/public-docs/reporting-api/) · [Support](#contact)

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
