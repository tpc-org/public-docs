---
layout: page
---

# Hola AI Ads — ads.txt & app-ads.txt Guide

**[ads.txt Guide](/public-docs/ads-txt/)** · [Web Integration Guide](/public-docs/publisher-integration/) · [Mobile SDK Guide](/public-docs/mobile-sdk-integration/) · [Server-side Guide](/public-docs/server-side-integration/) · [Reporting API Guide](/public-docs/reporting-api/) · [Payment Details Guide](/public-docs/payment-details/) · [Support](#contact)

---

Advertisers and demand partners check `ads.txt` (websites) and `app-ads.txt`
(mobile apps) to confirm that Hola AI is authorised to sell your ad inventory.
If the file is missing or doesn't list Hola AI, buyers can lower their bids
or stop bidding on your inventory altogether.

Rather than copying our lines into your own file and keeping them up to date
by hand, we recommend a **redirect**. You point your `/ads.txt` (and/or
`/app-ads.txt`) at the file Hola AI hosts:

```
https://sayhola.ai/ads.txt
```

Ad-system crawlers follow the redirect and read our file as if it were yours.
When Hola AI adds or changes a demand partner, your site picks up the change
automatically and you don't have to do anything.

## Which file do you need?

| You publish… | File | Where it must live |
|---|---|---|
| A website | `ads.txt` | Root of your site's domain: `https://example.com/ads.txt` |
| A mobile app (App Store / Google Play) | `app-ads.txt` | Root of the **developer website** listed on your app's store page: `https://example.com/app-ads.txt` |
| Both | Both | Set up both redirects. They can point to the same target. |

Both files redirect to the same place: **`https://sayhola.ai/ads.txt`**.

Server-side integrations (your backend calling Hola AI directly, no browser
or app) don't need either file.

### Root domain, not subdomain

Crawlers look for the file on your **root domain**. For ads on
`chat.example.com`, the crawler checks `https://example.com/ads.txt`. Set the
redirect up on the host that serves `example.com` (and `www.example.com` if
that serves your site too). Setting it up only on a subdomain isn't enough.

For apps, the domain is whatever appears as the developer website in your
App Store / Google Play listing. If the listing says
`https://www.example.com`, the crawler fetches
`https://example.com/app-ads.txt`.

## Redirect rules (read before configuring)

The IAB ads.txt specification allows any number of redirects **within your
own domain**, but only **one** redirect to another domain. If the chain hops
domains twice, the crawler treats the file as an error, which is the same as
having no file at all.

1. **Use the exact target `https://sayhola.ai/ads.txt`.** Use `https://`, no
   `www.`, and no trailing slash. `http://sayhola.ai/ads.txt` and
   `https://www.sayhola.ai/ads.txt` both redirect again, which makes a second
   cross-domain hop and breaks the chain.
2. **Use a `301` (permanent) redirect.** The spec allows `301`, `302` and
   `307`. Don't use `308`, because it isn't on the spec's list and some
   crawlers won't follow it.
3. **Match the path exactly.** Redirect `/ads.txt` only, not every path
   starting with `/ads.txt`.
4. **Don't stack redirects.** If `/ads.txt` already redirects somewhere else,
   *replace* that rule rather than adding another one.
5. **Same-domain hops first are fine.** `http://example.com/ads.txt` →
   `https://example.com/ads.txt` → `https://sayhola.ai/ads.txt` is valid,
   because only the last hop leaves your domain.
6. **Delete or rename any existing `ads.txt` file** on your server. That way
   nobody later edits a file that's never served, and an ads.txt plugin
   can't serve it instead of the redirect.

The examples below use `example.com`. Replace it with your own domain. If you
only need one of the two files, leave out the other line.

## Apache

Add the following to your site's `<VirtualHost>` block for both port 80 and
443, or to the `.htaccess` file in your document root. It uses `mod_alias`,
which is enabled on almost every Apache install:

```apache
RedirectMatch 301 ^/ads\.txt$     https://sayhola.ai/ads.txt
RedirectMatch 301 ^/app-ads\.txt$ https://sayhola.ai/ads.txt
```

Use `RedirectMatch` with the anchored pattern rather than plain `Redirect`.
`Redirect` is a prefix match, so it would also catch paths like
`/ads.txt/anything`.

**WordPress and other sites using `mod_rewrite` in `.htaccess`:** add the
rules as `RewriteRule`s **above** the `# BEGIN WordPress` block. That way they
run before WordPress (or an ads.txt plugin) handles the request:

```apache
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteRule ^ads\.txt$     https://sayhola.ai/ads.txt [R=301,L]
RewriteRule ^app-ads\.txt$ https://sayhola.ai/ads.txt [R=301,L]
</IfModule>

# BEGIN WordPress
...
```

If you use an ads.txt plugin (for example "Ads.txt Manager"), deactivate it
or clear its contents so it doesn't compete with the redirect.

Changes to `.htaccess` take effect immediately. After editing a
`<VirtualHost>`, check the config and reload:

```bash
sudo apachectl configtest && sudo systemctl reload apache2   # httpd on RHEL/Amazon Linux
```

## nginx

Add exact-match `location` blocks inside **each** `server { … }` block that
serves your root domain (and `www.`, if separate):

```nginx
location = /ads.txt {
    return 301 https://sayhola.ai/ads.txt;
}

location = /app-ads.txt {
    return 301 https://sayhola.ai/ads.txt;
}
```

The `=` modifier makes it an exact match, and nginx checks exact-match
locations before any `location /` or `try_files` catch-all your app uses. So
these rules work even in front of a single-page app or framework router.

If your port-80 `server` block already does a blanket
`return 301 https://$host$request_uri;`, you only need these rules in the
port-443 block. The HTTP→HTTPS hop stays on your own domain, which is
allowed.

Check the config and reload:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

## Cloudflare

These steps only apply if your domain's traffic is **proxied** through
Cloudflare (orange cloud on the DNS record for your root domain, and `www`
if used). If you only use Cloudflare for DNS ("DNS only" / grey cloud),
Cloudflare never sees the request. In that case, set up the redirect on your
origin server using the [Apache](#apache) or [nginx](#nginx) instructions
instead.

### Option A — Redirect Rule (recommended)

1. In the Cloudflare dashboard, select your domain, then go to **Rules →
   Overview → Create rule → Redirect Rule**.
2. **Rule name:** `ads.txt → Hola AI`
3. **If incoming requests match…** choose **Custom filter expression** and set:
   - Field: **URI Path**, Operator: **equals**, Value: `/ads.txt`

   Use **equals**, not *starts with* or *contains*. To cover `app-ads.txt` in
   the same rule, click **Or** and add a second condition: URI Path equals
   `/app-ads.txt`. The resulting expression (viewable via **Edit
   expression**) is:

   ```
   (http.request.uri.path eq "/ads.txt") or (http.request.uri.path eq "/app-ads.txt")
   ```

   If several hostnames on the zone serve different sites and you only want
   this on some of them, add `and http.host in {"example.com" "www.example.com"}`.
4. **Then…**
   - Type: **Static**
   - URL: `https://sayhola.ai/ads.txt`
   - Status code: **301**
   - **Preserve query string:** leave unchecked
5. Click **Deploy**.

Redirect Rules run at Cloudflare's edge before the request reaches your
origin, so whatever your server would have returned for `/ads.txt` is
ignored.

> If you already have a Cloudflare **Page Rule** forwarding `/ads.txt`, delete
> it once the Redirect Rule is in place. Page Rules are deprecated, and two
> rules on the same path make it hard to tell which one is responding.

### Option B — Cloudflare Pages or Workers static assets

If your site is deployed on Cloudflare Pages (or Workers with static assets),
add these lines to the `_redirects` file in your build output directory
(create the file if it doesn't exist), then redeploy:

```
/ads.txt      https://sayhola.ai/ads.txt  301
/app-ads.txt  https://sayhola.ai/ads.txt  301
```

Always set the status code explicitly. Pages defaults to `302` when it's
left off. Redirects apply even if a static `ads.txt` file still exists in
the output, but delete it anyway so nobody edits a file that's never served.

## Verify the redirect

Run this for each file and each hostname you serve (`example.com`,
`www.example.com`, and the `http://` variants):

```bash
curl -sI https://example.com/ads.txt | grep -iE '^HTTP|^location'
```

Expected:

```
HTTP/2 301
location: https://sayhola.ai/ads.txt
```

Then confirm the full chain resolves to the file's contents:

```bash
curl -sL https://example.com/ads.txt
```

You should see the same lines as
[https://sayhola.ai/ads.txt](https://sayhola.ai/ads.txt). For `app-ads.txt`,
run the same two commands with `/app-ads.txt`.

To check every hop, use `curl -sIL https://example.com/ads.txt | grep -iE '^HTTP|^location'`.
Exactly one `location:` header should point off your domain, and it must be
`https://sayhola.ai/ads.txt`, followed by a final `200`.

Crawlers re-check the file on their own schedule, typically within a few
days. You don't need to submit it anywhere.

## Already have an ads.txt file?

A redirect **replaces** your file. Crawlers see only Hola AI's lines, never
your own. If you already list other ad partners (your own Google AdSense or
Ad Manager account, another SSP, a direct deal), **don't redirect**. Your
existing partners would stop being authorised and their demand would drop.

Instead, add Hola AI's lines to your existing file. Copy the current
contents of [https://sayhola.ai/ads.txt](https://sayhola.ai/ads.txt) and
paste them below your existing entries. Ask your Hola AI account manager to
let you know when those lines change, since you'll need to update your copy
by hand.

## Troubleshooting

### `curl` shows a 200 with my old file, not a 301

Something earlier in the stack is serving a real file before your rule
runs. Look for a physical `ads.txt` in your document root, an ads.txt plugin
(WordPress), a CDN cache in front of the origin (purge `/ads.txt`), or a
Cloudflare DNS record set to "DNS only", which means a Redirect Rule never
runs.

### Redirect works but a partner still reports the file as invalid

Run `curl -sIL` (see [Verify the redirect](#verify-the-redirect)) and count
the cross-domain hops. A chain that ends at `http://sayhola.ai/…` or
`https://www.sayhola.ai/…` is two hops off your domain, so fix the target
URL. Also confirm the status code isn't `308`.

### My ads run on a subdomain only

The redirect still has to be on the root domain (`example.com/ads.txt`), not
just the subdomain. See [Root domain, not subdomain](#root-domain-not-subdomain).

### My app's store listing has no developer website

`app-ads.txt` requires a developer website on your App Store / Google Play
listing. Add one (a domain you control), then set up the redirect on that
domain.

## Contact

For questions about ads.txt, app-ads.txt, or which lines your domain needs,
contact your Hola AI account manager.
