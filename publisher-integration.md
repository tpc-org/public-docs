---
layout: page
---

# Hola AI Ads — Publisher Integration Guide

**[Integration Guide](external-integration.md)** · [Changelog](#) · [Support](#contact)

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

### Live format examples

See [Ad Format Examples](ad-format-examples.html) for rendered previews of each format
using the default Hola AI creative styling.

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
    userId:   'user-123',     // optional — bundle auto-generates if omitted
    chatId:   'conv-abc'      // optional — bundle auto-generates if omitted
  };
</script>
```

`messages` is the only field that changes ad behaviour. `userId` and `chatId` are
auto-generated when omitted — supply them only if you want to pass stable identifiers.

**You do not need to pass any bidder parameters.** The bundle and server-side
configuration handle all demand partner communication automatically.

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

If your site captures consent during publisher sign-up or another flow, you can pass a
TCF consent string directly:

```html
<script>
  window.tpc = window.tpc || {};
  window.tpc.settings = {
    tcfString: '<your-tcf-consent-string>'
  };
</script>
```

The bundle will use this string in place of a CMP. If neither a CMP nor a `tcfString`
is provided, the bundle proceeds with default consent settings as configured by your
Hola AI account manager.

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
> [iabtcf-encoder](https://www.npmjs.com/package/@iabtcf/encoder).

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

## Performance notes

- The bundle loads **asynchronously** (`<script async>`) — it does not block page rendering
- Auctions start as soon as the bundle parses, overlapping with the rest of the page load
- Default bid timeout: 1500ms
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
