---
layout: page
---

# Publisher integration guide

This guide is for developers integrating Hola AI Ads into a publisher
page. Hola AI Ads is a header bidding ad service built on open standards
(Prebid and OpenRTB). You'll need two things: a `<script>` tag and one
or more `<div>` elements.

## Quick start

Add this to your page's `<head>`:

```html
<script async src="https://s3.tpcsrv.com/clients/<your-client-id>/prebid.js"></script>
```

Then place divs where you want ads to appear:

```html
<!-- Hola AI Ad Placement -->
<div id="<placement-name>"></div>
```

That's it. The bundle handles everything else — auction, render,
failsafe. Add only the divs you want; unused placements don't run.

Your Hola AI account manager will provide:
- Your specific bundle URL
- Your placement tags, pre-configured with your preferred ad formats and display settings

## Ad formats

Hola AI Ads supports display (banner), native, and outstream video
formats. Your account manager will configure your placements and provide
a tag or tags for each format you want to run. Each placement tag is
pre-configured with the appropriate format, size, and display settings
for your site.

To add a placement to your page:

```html
<!-- Hola AI Ad Placement -->
<div id="<placement-name>"></div>
```

Place as many divs as you have placements. Unused placements — divs
absent from the page — don't fire.

## CMP requirement

Your page must have an IAB TCF v2.2-compatible Consent Management
Platform (CMP) loaded before (or alongside) the Hola AI bundle. It is
your responsibility to ensure appropriate consent signals are provided
before the auction runs.

If you do not have a CMP, contact your Hola AI account manager. Use of
the platform is subject to applicable privacy and data protection laws
and regulations, including GDPR, CCPA/CPRA, and similar frameworks.
Specific responsibilities and obligations related to compliance, data
handling, and processing are defined in the applicable Data Processing
Agreement (DPA) between the parties.

## Page integration example

A complete minimal page:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>My page</title>

  <!-- Your CMP script goes here -->

  <!-- Hola AI bundle (replace with your actual bundle URL) -->
  <script async src="https://s3.tpcsrv.com/clients/<clientid>/prebid.js"></script>
</head>
<body>

  <!-- Your page content -->

  <!-- Hola AI Ad Placements — add only the placements you want -->
  <div id="<placement-name>"></div>

</body>
</html>
```

## Performance notes

- The bundle loads **asynchronously** (`<script async>`) — it does not
  block page rendering
- Auctions start as soon as the bundle parses, overlapping with the
  rest of the page load
- Default bid timeout is 1500ms; bids that don't return in time are
  dropped
- A 3500ms failsafe ensures render always completes even if the auction
  hangs

For best performance, place the `<script>` tag in `<head>` (with
`async`). Placing it at the end of `<body>` delays auction start.

## Troubleshooting

### No ads appearing

1. Check your browser console for errors — the bundle logs warnings
   with the `[tpc]` prefix
2. Confirm the div `id` exactly matches your placement code (case-sensitive)
3. Check the Network tab for a POST to your Hola AI auction endpoint
   — if it's missing, the bundle didn't load
4. Check your CMP is firing before the bundle's timeout expires

### Ads appearing but wrong size

The ad renders at the size configured for your placement. If your
container is smaller than the creative, the iframe clips. Ensure your
container meets the minimum dimensions provided by your account manager.

### Native ad not styled the way I want

The native creative renders inside an iframe, so parent-page CSS cannot
reach it. Contact your Hola AI account manager to customise the native
creative template (fonts, colours, layout, border).

### Testing without real traffic

To test the integration before going live, open your page with
`?pbjs_debug=true` appended to the URL. This enables verbose Prebid
logging in the browser console showing every auction event.

## Contact

For integration questions, placement provisioning, or to request
additional formats, contact your Hola AI account manager.
