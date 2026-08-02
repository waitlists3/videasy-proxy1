# videasy-proxy

A tiny **Cloudflare Worker** that reverse-proxies [`player.videasy.to`](https://player.videasy.to) and **strips the `ab.js` ad script**. You get the full Videasy player on your own `*.workers.dev` URL, ad-script removed, embeddable anywhere.

```
https://<your-worker>.workers.dev/tv/1434/1/1   →  player.videasy.to/tv/1434/1/1
https://<your-worker>.workers.dev/movie/550     →  player.videasy.to/movie/550
```

## Deploy

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/fartboblover3/videasy-proxy)

Click the button, connect your Cloudflare account, and it deploys straight from this repo — no local setup. When it's done you'll get a `https://videasy-proxy.<your-subdomain>.workers.dev` URL.

## Usage

Once deployed, use it exactly like the Videasy player, but on your worker URL:

```html
<!-- Movie -->
<iframe src="https://your-worker.workers.dev/movie/550" allowfullscreen></iframe>

<!-- TV episode: /tv/{tmdbId}/{season}/{episode} -->
<iframe src="https://your-worker.workers.dev/tv/1434/1/1" allowfullscreen></iframe>
```

Any path works — the worker proxies whatever you request to the same path on `player.videasy.to`.

## What it does

- **Proxies *everything*** — not just `player.videasy.to`, but every cross-domain resource the player pulls in: third-party scripts, API calls, and the **video stream itself** (from whatever CDN it lives on). Nothing leaks to an external domain in the viewer's browser; it all flows through your worker.
  - Same-origin player requests hit the worker and forward to `player.videasy.to`.
  - Cross-domain URLs are routed as `/_ext/<encoded url>`.
  - A tiny shim injected into the page reroutes **runtime** requests too — `fetch`, `XMLHttpRequest`, `EventSource`, `sendBeacon`, and dynamically-added `<script>`/`<img>`/`<source>`/`<video>`/`<link>` elements. Since hls.js / dash pull the stream via fetch/XHR, the stream is proxied as well.
- **Removes the `ab.js` ad script** — the `<script>` tag is stripped from the HTML, any direct request for `ab.js` returns an empty response, and the shim drops it if it's injected at runtime. Every other script loads normally.
- **Drops `Content-Security-Policy` / `X-Frame-Options`** and adds permissive CORS, so the player embeds cleanly in an iframe from any site.

> Note: rerouting a live web app through a proxy is best-effort — most requests (page, scripts, API, HLS/DASH stream) are covered, but an app can always construct a request in a way a shim doesn't intercept. If a specific stream or feature misbehaves, tell me the network request that leaked and I'll extend the shim.

## Configuration

Everything lives in [`src/index.js`](src/index.js). Two constants at the top control it:

```js
const UPSTREAM_HOST = "player.videasy.to"; // default site to proxy
const AD_NAMES = ["ab.js"];                // script filenames to block (add more here)
```

Change `UPSTREAM_HOST` to proxy a different site. Add filenames to `AD_NAMES` to block more scripts — anything whose filename matches is dropped, no matter which domain it comes from.

To rename the worker (and its URL), edit `name` in [`wrangler.toml`](wrangler.toml).

## Local development

```bash
npm install
npm run dev        # runs the worker locally at http://localhost:8787
# then open http://localhost:8787/movie/550
```

Deploy manually (instead of the button) with:

```bash
npx wrangler login
npm run deploy
```

## Disclaimer

This proxies a third-party service you do not control; it does not host or store any media. You are responsible for how you deploy and use it and for complying with the terms of the upstream service and the laws in your jurisdiction. Provided as-is, no warranty.
