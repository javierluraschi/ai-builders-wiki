---
title: "Why a deploy looks broken in one browser but fine in another"
description: "A new HTML page paired with an old cached CSS or JS file: how to spot it, and how to give assets a URL that changes when they do."
category: Guides
tags: [hosting, basics]
order: 2
checked: 2026-11-09
---

<!-- NAV:START -->
[Home](README.md) › [Guides](Category-Guides.md) · Tags: [hosting](Tag-hosting.md), [basics](Tag-basics.md)
<!-- NAV:END -->

# Why a deploy looks broken in one browser but fine in another

**If a freshly deployed page looks wrong in one browser tab but right in a private window, the server is probably fine and the browser is running the previous copy of a CSS or JavaScript file.** The HTML came from the new deploy; the stylesheet came from the cache the browser filled before it. The fix is to serve each file from a URL that changes when the file's contents change.

This is common when a site is assembled by hand — a small server rendering templates, a static folder behind a container — rather than by a framework or static host that fingerprints assets for you.

## What the mismatch looks like

The two files arrive from different deployments, so what you see is one half of each:

- Text you added at the top of a page pushes the rest of the layout around, but the stylesheet still targets the old structure.
- A style change "does not apply", yet the developer tools show the new rule, or the old one wins because two rules target the same element.
- A layout bug that was fixed is still visible, but only in the window that was already open before the deploy.

The tell is the comparison: the normal window is broken; a private or incognito window, which starts with an empty cache, is correct. A hard refresh usually fixes the window for now — and that is exactly the trap, because the next visitor with an old cache sees the broken version again.

## Check what the browser actually received

1. Open the developer tools' **Network** panel, reload, and look at the CSS and JS requests. A request served from the browser's disk or memory cache, or answered `304 Not Modified`, is a candidate.
2. Compare the two windows side by side, or fetch the files directly:

```sh
# What the server publishes, without any browser cache in the way
curl -sI https://example.com/styles.css

# Ask for a URL the browser has never requested
curl -sI "https://example.com/styles.css?cachecheck=$(date +%s)"
```

If the first command's file is the old one, the deploy itself did not roll out — check the build status and the running instance's logs before changing anything about caching. If the first is new but the page still uses an old copy, the browser is holding a copy it was told it could keep.

3. Look at `Cache-Control` in the response. `max-age=3600`, for example, tells the browser it may reuse the file for an hour without asking the server ([MDN, Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control), checked 2026-11-09). HTML is often served with a short lifetime or revalidated on every request, which is why the page can be new while its stylesheet is not.

## The fix: make the URL change with the file

A browser can cache `styles.css` for an hour safely only if that URL's contents stay the same. Once the contents change, the URL must change too. Every produced file gets a version of its own:

```html
<link rel="stylesheet" href="/styles.css?v=20261109-1">
<script src="/app.js?v=20261109-1" defer></script>
```

The version can be a build number, a short commit hash, or the file's modification time. The important rule: **the marker changes when the file changes, and stays the same when it does not**, so unchanged assets keep their long cache. Many build tools instead put a content hash in the file name (`styles.9f2c1a.css`); the effect is the same.

If the server renders the HTML, compute the version once at boot and use it in the template, so nobody has to remember to bump it:

```js
const fs = require('fs');
const path = require('path');

const assetVersion = ['styles.css', 'app.js']
  .map((file) => fs.statSync(path.join('public', file)).mtimeMs)
  .reduce((a, b) => Math.max(a, b), 0);

// in the template:
// <link rel="stylesheet" href="/styles.css?v=${assetVersion}">
```

Then keep the HTML's own cache short — or revalidate it — so the next page load points at the new versions. The HTML is the index that says which CSS and JS belong to this deploy; a long-cached index pins visitors to an old, internally consistent set.

## After a deploy, check from a clean cache

- Open the site in a private window, or empty the cache and disable it in the developer tools' Network panel while testing.
- Confirm the HTML references the new asset URL, and that the asset URL returns the new file.
- Hard-refresh (or bypass the cache) in the window that looked broken. If it still looks wrong, the problem is not caching.
- When the site sits behind a CDN, purge its copy of the HTML as well; purging one without the other recreates the same mismatch at the CDN.

## When it is not the cache

- **The deploy did not reach the running instance.** A successful build is not the same as a restarted app; check the app's status and logs, and that the request is served by the new version.
- **A service worker is in control.** A service worker can serve old files even after a hard refresh; unregister it for the site while testing, and version its cache the same way.
- **Two instances are behind the load balancer.** One still runs the old image; the page alternates between versions. Finish the rollout instead of debugging the browser.

## Related pages

- [From idea to deployed AI app](Idea-to-deployed-AI-app.md): the deploy step, and how to tell a build failure from an app failure
- [The cloud, for AI apps](Cloud-for-AI-apps.md): containers and what runs inside them

## See also

<!-- SEEALSO:START -->
Glossary terms used on this page: [Container](Glossary.md#container), [Deploy](Glossary.md#deploy), [Host](Glossary.md#host), [Logs](Glossary.md#logs), [Tool](Glossary.md#tool).
<!-- SEEALSO:END -->

## Sources

All checked on 2026-11-09.

- MDN, [Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control): `max-age` and how long a response may be reused without asking the server
- MDN, [HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching): fresh vs stale responses, revalidation, and changing the URL as a way to bust a cache
- IETF, [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html): freshness, validation, and cache keys

---

First version written with AI assistance, from a deploy where the new HTML and the browser's old stylesheet combined into an overlapping page layout, and checked against the [sources](#sources) on 2026-11-09. Found a mistake? [Fix it or open an issue](CONTRIBUTING.md). Text licensed under [CC BY 4.0](LICENSE).
