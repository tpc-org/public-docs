---
layout: page
---

# Hola AI Ads — Publisher Integration Guide

**[Integration Guide](/public-docs/publisher-integration/)** · [Mobile SDK Guide](/public-docs/mobile-sdk-integration/) · [Server-side Guide](/public-docs/server-side-integration/) · [Reporting API Guide](/public-docs/reporting-api/) · [Payment Details Guide](/public-docs/payment-details/) · [Changelog](#) · [Support](#contact)

---

This guide is for developers integrating Hola AI Ads into a publisher page.
Hola AI Ads is a header bidding ad service built on open standards (Prebid and OpenRTB).
You'll need two things: a `<script>` tag and one `<div>` element per placement.

## Quick start

Add this to your page's `<head>`:

```html
<script async src="https://s3.tpcsrv.com/clients/<your-client-id>/prebid.js"></script>
```

Then place a div where you want an ad to appear:

```html
<!-- Hola AI Ad Placement -->
<div id="<placement-id>"></div>
```

That's it. The bundle handles everything else — auction, format selection, creative
rendering, countdown, and refresh. Your Hola AI account manager will provide your
specific bundle URL and placement IDs.

## Ad formats

Hola AI Ads runs display (banner), native, and outstream video formats from a single
placement. The highest-value format wins each auction and is rendered automatically.
No per-format configuration is needed on the publisher side.

### Native

A sponsored card with title, body text, optional thumbnail, and call-to-action.
Renders inline without an iframe.

```
┌──────────────────────────────────┐
│ Brand Name               Sponsored│
│ ┌──────┐ Ad headline here         │
│ │      │ Short description of the  │
│ │ img  │ product or service.       │
│ └──────┘ [Learn More]             │
└──────────────────────────────────┘
```

### Display (banner)

Standard 300×250 banner rendered in an iframe.

### Video (outstream)

640×480 outstream video player. Plays inline; no pre-roll or page video required.

### Format examples

<table><tr><td valign="top" width="340">

**Native**

<div style="font-family:-apple-system,sans-serif;max-width:300px;border:1px solid #e8e8e8;border-radius:8px;overflow:hidden;box-shadow:0 1px 6px rgba(0,0,0,.07)">
  <div style="height:3px;background:#0070c0;width:70%"></div>
  <div style="padding:14px 12px 12px">
    <div style="display:flex;align-items:center;gap:8px;margin-bottom:10px">
      <span style="font-size:12px;font-weight:600;color:#333;flex:1">Brand Name</span>
      <span style="font-size:10px;color:#ccc">Sponsored</span>
    </div>
    <div style="display:flex;gap:10px;align-items:flex-start">
      <div style="width:80px;height:80px;background:#e8eef5;border-radius:4px;flex-shrink:0"></div>
      <div style="flex:1">
        <div style="font-size:13px;font-weight:600;color:#222;margin-bottom:6px">Ad headline goes here</div>
        <div style="font-size:11px;color:#666;margin-bottom:10px">Short description of the product or service being promoted.</div>
        <span style="font-size:11px;color:#fff;background:#0070c0;padding:4px 12px;border-radius:12px">Learn More</span>
      </div>
    </div>
  </div>
</div>

</td><td valign="top" width="340">

**Display (banner) — 300×250**

<div style="font-family:-apple-system,sans-serif;max-width:300px;border:1px solid #e8e8e8;border-radius:8px;overflow:hidden;box-shadow:0 1px 6px rgba(0,0,0,.07)">
  <div style="height:3px;background:#0070c0;width:70%"></div>
  <div style="width:300px;height:250px;background:linear-gradient(135deg,#e8eef5,#d0dce8);display:flex;flex-direction:column;align-items:center;justify-content:center;gap:12px;position:relative">
    <span style="position:absolute;top:6px;right:8px;font-size:10px;color:#aaa">Sponsored</span>
    <div style="font-size:16px;font-weight:700;color:#222;text-align:center;padding:0 20px">Banner creative</div>
    <div style="font-size:12px;color:#555;text-align:center;padding:0 20px">300×250 display ad</div>
    <span style="font-size:12px;color:#fff;background:#0070c0;padding:7px 18px;border-radius:20px">Shop Now</span>
  </div>
</div>

</td></tr><tr><td valign="top" colspan="2">

**Video (outstream) — 640×480, scales to container width**

<div style="font-family:-apple-system,sans-serif;max-width:300px;border:1px solid #e8e8e8;border-radius:8px;overflow:hidden;box-shadow:0 1px 6px rgba(0,0,0,.07)">
  <div style="height:3px;background:#0070c0;width:70%"></div>
  <div style="width:300px;height:200px;background:#111;display:flex;flex-direction:column;align-items:center;justify-content:center;gap:10px;position:relative">
    <span style="position:absolute;top:6px;right:8px;font-size:10px;color:rgba(255,255,255,.5)">Sponsored</span>
    <div style="width:52px;height:52px;border-radius:50%;border:2px solid rgba(255,255,255,.6);display:flex;align-items:center;justify-content:center">
      <div style="border-style:solid;border-width:10px 0 10px 18px;border-color:transparent transparent transparent rgba(255,255,255,.9);margin-left:3px"></div>
    </div>
    <span style="font-size:12px;color:rgba(255,255,255,.5)">Outstream video ad</span>
  </div>
</div>

</td></tr></table>

## Contextual AI ads

If your site is an AI assistant or chat interface, passing conversation context improves
ad relevance and yield.

### Without context (default)

When no context is provided, the placement selects ads without conversation-specific
targeting. No additional setup is needed.

### With context

When `messages` are provided, the placement switches to contextual mode — ads are
matched to the current conversation topic.

Set `window.tpc.data` before or after the bundle script:

```html
<script>
  window.tpc = window.tpc || {};
  window.tpc.data = {
    messages: [               // required for contextual mode
      { role: 'user', content: "I'm looking for new shoes" }
    ],
    userId:      'user-123',  // optional — bundle auto-generates if omitted
    chatId:      'conv-abc',  // optional — bundle auto-generates if omitted
    hashedEmail: '9f86d0...'  // optional — logged-in users only, see below
  };
</script>
```

`messages` is the only field that changes ad behaviour. `userId` and `chatId` are
auto-generated when omitted — supply them only if you want to pass stable identifiers.

**You do not need to pass any bidder parameters.** The bundle and server-side
configuration handle all demand partner communication automatically.

### Logged-in users (optional)

If a visitor is logged in and you know their email address, you can pass a
**hashed** email to improve ad matching for one of our two demand partners:

```html
<script>
  window.tpc.data = {
    messages: [{ role: 'user', content: "..." }],
    hashedEmail: '<sha256 hex digest>'
  };
</script>
```

- **Hash it yourself, before it ever reaches this bundle.** Required algorithm:
  `SHA-256(email.trim().toLowerCase())`, as a lowercase hex string. Never pass a
  plain email address — the field is named `hashedEmail` deliberately, and an
  unhashed value will not be recognized as an email by our demand.
- **Only some demand actually uses this signal.** Not every demand source
  supports (or permits) identity matching via email, hashed or not — our
  server forwards it only where it's applicable, and silently ignores it
  otherwise. There's nothing you need to know about which is which; set
  it whenever you have it, and omit it otherwise.
- Entirely optional. Omitting it changes nothing else about ad behavior —
  it's purely an additional identity signal where supported.

## Publisher settings

Override default bundle behaviour via `window.tpc.settings`:

```html
<script>
  window.tpc = window.tpc || {};
  window.tpc.settings = {
    adDisplaySeconds: 20,   // how long each ad is shown (default: 30)
    adMaxRefresh: 2         // max sequential ads per session (default: 3)
  };
</script>
```

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `adDisplaySeconds` | number | 30 | Seconds each ad is displayed before auto-closing |
| `adMaxRefresh` | number | 3 | Maximum number of sequential ads shown per session |

Settings can be set at any time before the first auction fires. They are read lazily
so placement order relative to the bundle script does not matter.

> **Coming soon:** `renderMode: 'iframe'` — renders the creative inside an iframe for
> additional isolation. Available on request.

## Consent

### With a CMP (recommended)

The bundle integrates with any IAB TCF 2.x-compatible Consent Management Platform.
Load your CMP before or alongside the bundle — consent signals are read automatically.

### Without a CMP

If your site captures consent during publisher sign-up or another flow, you can pass
consent strings directly via `window.tpc.settings`:

```html
<script>
  window.tpc = window.tpc || {};
  window.tpc.settings = {
    tcfString: '<tcf-2.3-consent-string>',   // GDPR / EU
    gppString: '<gpp-consent-string>',        // US state privacy (GPP)
    gppSid:    [7]                            // GPP section IDs — 7 = USNAT
  };
</script>
```

The bundle uses these strings in place of a CMP. Set only the strings relevant to your
jurisdiction — `tcfString` for GDPR markets, `gppString`/`gppSid` for US markets. If
neither is provided, the bundle proceeds with default consent settings as configured by
your Hola AI account manager.

#### Example TCF 2.3 strings (for testing only)

**All purposes consented, all vendors:**
```
CQJz4oAQJz4oAGXABBENBdFsAP_gAEPgAAATIIDoBJCoAAAAAA
```

**No consent:**
```
CQAAAAAAAAAAAAAAAABzAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

> These strings are illustrative only. For production use, generate strings with your
> CMP or a TCF-compliant encoder library such as
> [@iabtcf/encoder](https://www.npmjs.com/package/@iabtcf/encoder).

#### Example GPP strings — USNAT (section 7) (for testing only)

**All sharing permitted (no opt-outs):**
```
DBABBg==
```

**All opt-outs set (sharing not permitted):**
```
DBABjw==
```

Pass alongside `gppSid: [7]` to identify the USNAT section:

```js
window.tpc.settings = {
  gppString: 'DBABBg==',
  gppSid:    [7]
};
```

> These strings are illustrative only. For production use, generate strings with your
> CMP or a GPP-compliant encoder library such as
> [@iabtcf/encoder](https://www.npmjs.com/package/@iabtcf/encoder) (supports GPP).

## Page integration example

A complete minimal page:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>My page</title>

  <!-- Publisher settings (optional) -->
  <script>
    window.tpc = {
      settings: { adDisplaySeconds: 30, adMaxRefresh: 3 },
      data: {
        messages: [{ role: 'user', content: "I'm looking for new shoes" }]
      }
    };
  </script>

  <!-- Hola AI bundle (replace with your actual bundle URL) -->
  <script async src="https://s3.tpcsrv.com/clients/<clientid>/prebid.js"></script>
</head>
<body>

  <!-- Your page content -->

  <!-- Hola AI Ad Placement -->
  <div id="<placement-id>"></div>

</body>
</html>
```

## Agency / multi-domain integration

If you operate multiple publisher domains under one agency, all of them can
share **one bundle URL** — you don't need a separate `<script>` tag per
domain. Each domain just gets its **own placement `<div>` id**; the shared
bundle detects which one is present on the page and runs that domain's
auction automatically.

**Site A** (e.g. `site-a.example.com`):

```html
<script async src="https://s3.tpcsrv.com/clients/<your-agency-id>/prebid.js"></script>
<div id="<site-a-placement-id>"></div>
```

**Site B** (e.g. `site-b.example.com`):

```html
<script async src="https://s3.tpcsrv.com/clients/<your-agency-id>/prebid.js"></script>
<div id="<site-b-placement-id>"></div>
```

Notice the `<script>` tag is **identical** on both sites — only the div id
differs. Your Hola AI account manager provisions one placement (and one
placement ID) per domain under your agency, and provides the resulting
`<div id="...">` for each site to paste in. Add as many domains as you
operate; each one only ever needs its own single div, never any other
domain's.

## Reporting API

Want your revenue data programmatically instead of (or alongside) the
dashboard UI? See the [Reporting API Guide](/public-docs/reporting-api/) —
a simple REST API authenticated with an API key your account manager can
issue you, covering the same summary/timeseries/breakdown data shown in
the dashboard.

## Performance notes

- The bundle loads **asynchronously** (`<script async>`) — it does not block page rendering
- Auctions start as soon as the bundle parses, overlapping with the rest of the page load
- Default bid timeout: 2500ms
- Failsafe timeout: 3500ms — render completes even if the auction hangs

For best performance, place the `<script>` tag in `<head>` with `async`.

## Troubleshooting

### No ads appearing

1. Check the browser console for errors
2. Confirm the div `id` exactly matches your placement ID (case-sensitive)
3. Check the Network tab for a POST to the Hola AI auction endpoint — if missing, the
   bundle did not load
4. Append `?pbjs_debug=true` to the URL for verbose Prebid logging in the console

### Native ad not styled correctly

Native creatives render directly into the page DOM. Contact your Hola AI account manager
to customise the native template (fonts, colours, layout).

### Banner ad wrong size

The ad renders at the size configured for your placement. Ensure your container is at
least as wide as the configured banner size. Contact your account manager if you need
different dimensions.

## Contact

For integration questions, placement provisioning, or to request additional formats,
contact your Hola AI account manager.
